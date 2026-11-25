---
title: "GitOps Without Git: Shipping RHACM Policies as OCI Artifacts"
description: "How I package Red Hat ACM policies as an OCI artifact in Quay and let Argo CD deploy them, so hub clusters never need to reach a Git server."
date: 2026-11-25 15:00:00 -0500
categories: [OpenShift, GitOps]
tags: [openshift, rhacm, argocd, gitops, oci, policy, compliance]
---

Every fleet I've worked on in financial services hits the same wall. The
platform team wants GitOps. Security doesn't want production clusters talking
to the Git server, and on the disconnected sites they can't anyway. What
those clusters *can* reach is the container registry, because they already
pull every image from it.

So stop fighting it. Put the config in the registry.

Since Argo CD 3.1 you can point an Application at an OCI artifact the same
way you'd point it at a Git repo. OpenShift GitOps 1.18 ships Argo CD 3.1, so
this works on a supported stack today. This post walks through how I use it
to deliver Red Hat Advanced Cluster Management (ACM) policies to a fleet.

The full working example is on GitHub:
[sumantripuraneni/rhacm-cluster-policies](https://github.com/sumantripuraneni/rhacm-cluster-policies).

## The flow

```text
Git repo (engineers work here)
   │  tar + oras push
   ▼
OCI artifact in Quay (versioned, e.g. 1.0.2)
   │  Argo CD repo-server pulls and unpacks it
   ▼
Kustomize + PolicyGenerator render ACM Policies
   │  Argo CD applies them to the hub
   ▼
ACM propagates them to every cluster labelled base-config=true
```

Git is still where people write and review changes. It just isn't something
the clusters depend on at runtime. The registry is the only thing the hub
talks to.

## Why I like this setup

- **No Git egress from clusters.** One less firewall exception to justify in
  every security review.
- **Works disconnected.** You already mirror images into a local registry for
  air-gapped clusters. The policy bundle is just one more artifact to mirror.
- **Real versions.** `1.0.2` is an immutable, pushed release, not "whatever
  `main` looked like when Argo CD last polled."
- **Access control you already have.** Quay robot accounts give you a
  push-only identity for CI and a read-only one for clusters.
- **One hub, many clusters.** Argo CD only ever talks to the hub. ACM does the
  fan-out with placement, so you aren't running Argo CD against 200 clusters.

## What's in the bundle

The repo is a small Kustomize tree with one `PolicyGenerator`:

```text
base/
├── kustomization.yaml
├── namespace.yaml
├── policygenerator.yaml
└── manifests/
    ├── operators/       # OperatorPolicy per operator
    │   ├── compliance/
    │   ├── gatekeeper/
    │   ├── openshift-virtualization/
    │   └── webterminal/
    └── configuration/   # plain manifests the operators need
        ├── compliance/
        ├── console_banner/
        ├── etcd/
        └── openshift-virtualization/
```

The trick is that you don't write ACM `Policy` YAML by hand. You write
ordinary Kubernetes manifests and let PolicyGenerator wrap them:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: base-policies-pg
policyDefaults:
  namespace: acm-policies-openshift-gitops
  orderManifests: true
  remediationAction: enforce
  complianceType: musthave
  placement:
    labelSelector:
      base-config: "true"
policies:
  - name: etcd
    manifests:
      - path: manifests/configuration/etcd
  - name: compliance
    manifests:
      - path: manifests/operators/compliance
      - path: manifests/configuration/compliance
  - name: openshift-virtualization
    manifests:
      - path: manifests/operators/openshift-virtualization
      - path: manifests/configuration/openshift-virtualization
  - name: console-banner
    manifests:
      - path: manifests/configuration/console_banner
```

Two choices here are worth calling out.

**Operator and its config live in one policy.** The `compliance` policy holds
both the operator install and the `ScanSettingBinding`. With
`orderManifests: true`, the config doesn't get applied until the operator is
in place and its CRDs exist. Split them into two policies and you're back to
"apply it twice and hope."

**Placement is a label, not a list.** Any cluster labelled
`base-config=true` gets the baseline. Onboarding a new cluster is one
`oc label`, not a pull request.

### Installing operators with OperatorPolicy

`OperatorPolicy` is far cleaner than wrapping a Subscription and an
OperatorGroup in a ConfigurationPolicy. It knows what an operator install
looks like, reports CSV health back to the hub, and can clean up properly:

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: compliance-operator-install
spec:
  complianceType: musthave
  remediationAction: enforce
  severity: critical
  upgradeApproval: Automatic
  removalBehavior:
    clusterServiceVersions: Delete
    customResourceDefinitions: Delete
    operatorGroups: DeleteIfUnused
    subscriptions: Delete
  subscription:
    channel: stable
    name: compliance-operator
    namespace: openshift-compliance
    source: redhat-operators
    sourceNamespace: openshift-marketplace
  operatorGroup:
    name: openshift-compliance-operator
    namespace: openshift-compliance
    targetNamespaces:
      - openshift-compliance
```

> `upgradeApproval: Automatic` is fine for a lab. In production I set it to
> `None` and control operator versions explicitly, so an upstream channel
> update can't roll across the whole fleet on its own.
{: .prompt-warning }

### One manifest, a different value per cluster: hub templates

The console banner is my favourite small win. One manifest, and every cluster
shows its own name and data centre at the top of the console:

```yaml
apiVersion: console.openshift.io/v1
kind: ConsoleNotification
metadata:
  name: cluster
spec:
  text: '{{hub (index .ManagedClusterLabels "cluster-name") hub}}'
  location: BannerTop
  color: '#000000'
  backgroundColor: '#888888'
```

`{{hub ... hub}}` templates are resolved on the hub using the managed
cluster's labels. It's the difference between "which cluster am I on?" and
knowing, which matters a lot when someone is about to run `oc delete`.

## Setting it up

### 1. Teach Argo CD to run PolicyGenerator

PolicyGenerator is a Kustomize plugin, and Argo CD's repo-server doesn't ship
it. The neat part: the binary is already on your hub, inside the ACM
subscription controller image. Copy it in with an init container:

```bash
IMAGE=$(oc get deploy multicluster-operators-hub-subscription \
  -n open-cluster-management \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
[ -n "$IMAGE" ] && echo "OK: $IMAGE" || echo "IMAGE IS EMPTY - stop here"
```

```yaml
# argocd-policygen-patch.yaml
spec:
  kustomizeBuildOptions: --enable-alpha-plugins
  repo:
    env:
      - name: KUSTOMIZE_PLUGIN_HOME
        value: /etc/kustomize/plugin
    initContainers:
      - name: policy-generator-install
        image: ${IMAGE}
        command: ["/bin/bash"]
        args:
          - -c
          - cp /policy-generator/PolicyGenerator-not-fips-compliant /policy-generator-tmp/PolicyGenerator
        volumeMounts:
          - name: policy-generator
            mountPath: /policy-generator-tmp
    volumeMounts:
      - name: policy-generator
        mountPath: /etc/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator
    volumes:
      - name: policy-generator
        emptyDir: {}
```

```bash
oc patch argocd openshift-gitops -n openshift-gitops \
  --type=merge --patch-file argocd-policygen-patch.yaml
oc -n openshift-gitops rollout status deploy/openshift-gitops-repo-server
```

Pulling the binary from the hub's own image means its version always matches
the ACM version you're running. No separate download to track.

### 2. Prepare the hub

Argo CD's application controller needs RBAC to manage policy objects, plus a
namespace for the policies bound to a ClusterSet so placement can find
clusters:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: global
  namespace: acm-policies-openshift-gitops
spec:
  clusterSet: global
```

The full ClusterRole and binding are in the
[repo README](https://github.com/sumantripuraneni/rhacm-cluster-policies#2-prepare-the-hub).
Keep that ClusterRole scoped to the policy API groups. Don't hand the Argo CD
controller `cluster-admin` just to get this working.

### 3. Two robot accounts in Quay

Create a private repo `rhacm-cluster-policies` and two robot accounts:

- `rhacm_policies_push`: **Write**, used by CI or by you
- `rhacm_policies_pull`: **Read**, used by Argo CD

Clusters can read releases. They can never publish one.

### 4. Package and push

```bash
COPYFILE_DISABLE=1 tar --no-xattrs --exclude '._*' --exclude '.DS_Store' \
  -czf acm-policies.tgz base

oras login quay.io -u '<quay-org>+rhacm_policies_push' \
  --registry-config ~/.config/oras/auth.json

oras push --registry-config ~/.config/oras/auth.json \
  quay.io/<quay-org>/rhacm-cluster-policies:1.0.0 \
  acm-policies.tgz:application/vnd.oci.image.layer.v1.tar+gzip
```

There's no base image here. It's a single tar.gz layer that Argo CD downloads
and unpacks. The `COPYFILE_DISABLE` and `--exclude '._*'` flags matter if you
build on a Mac: without them, macOS slips `._*` metadata files into the
archive and Kustomize tries to parse them as YAML.

### 5. Point Argo CD at the registry

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rhacm-policies-oci
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: oci
  url: oci://quay.io/<quay-org>/rhacm-cluster-policies
  username: <quay-org>+rhacm_policies_pull
  password: <pull-robot-token>
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: acm-policies
  namespace: openshift-gitops
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - list:
        elements:
          - name: base
            path: base
  template:
    metadata:
      name: 'acm-policies-{{.name}}'
    spec:
      project: default
      source:
        repoURL: oci://quay.io/<quay-org>/rhacm-cluster-policies
        targetRevision: 1.0.0
        path: '{{.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: acm-policies-openshift-gitops
```

Only `repoURL` looks different from a Git-based Application. Everything else
is the same Argo CD you already know.

I use an ApplicationSet with a list generator even though there's only one
bundle today. When you add `base`, `prod-overlay` and `dmz-overlay`, it's
one more list entry, not a new Application.

### 6. Label a cluster and watch it go

```bash
oc label managedcluster local-cluster base-config=true
oc -n local-cluster get policy -w
```

```console
NAME                                                   REMEDIATION ACTION   COMPLIANCE STATE   AGE
acm-policies-openshift-gitops.compliance               enforce              Compliant          3m
acm-policies-openshift-gitops.console-banner           enforce              Compliant          3m
acm-policies-openshift-gitops.etcd                     enforce              Compliant          3m
acm-policies-openshift-gitops.gatekeeper               enforce              Compliant          3m
acm-policies-openshift-gitops.openshift-virtualization enforce              Compliant          3m
acm-policies-openshift-gitops.webterminal              enforce              Compliant          3m
```

## Shipping a change

Edit, re-tar, push a new tag, bump `targetRevision`:

```bash
oras push --registry-config ~/.config/oras/auth.json \
  quay.io/<quay-org>/rhacm-cluster-policies:1.0.1 \
  acm-policies.tgz:application/vnd.oci.image.layer.v1.tar+gzip

oc -n openshift-gitops patch applicationset acm-policies --type=merge -p \
  '{"spec":{"template":{"spec":{"source":{"targetRevision":"1.0.1"}}}}}'
```

You can also push every release as both `1.0.x` and `latest`, point the
ApplicationSet at `latest`, and turn on automated sync with `selfHeal`.
That's what the repo does in its last step, and it's handy in a lab.

**I wouldn't run production on `latest`.** The whole point of moving config
into a registry is that a release is an immutable, named thing. Pin
production hubs to a version tag, or better, a digest, and promote by
changing that one line. Lab and dev hubs can track `latest`.

## Things that will bite you

**Compliance auto-remediation reboots nodes.** A `ScanSetting` with
`autoApplyRemediations: true` turns scan findings into MachineConfigs, and
MachineConfigs roll nodes. Pushed fleet-wide by a policy, that's a lot of
reboots you didn't schedule. Start with auto-apply off, review the
remediations, then turn it on per environment.

**Make sure the binding points at your ScanSetting.** A `ScanSettingBinding`
with `settingsRef.name: default` uses the operator's built-in `default`
ScanSetting, not the custom one you shipped next to it. Your schedule and
storage settings are silently ignored.

**Namespaces with special labels must exist first.** `openshift-compliance`
needs privileged pod security labels before the operator pods can start.
Create it on the hub setup side, or ship it in the policy ahead of the
operator.

**etcd encryption isn't instant.** Setting `aescbc` (or `aesgcm`) on the
`APIServer` kicks off a re-encryption of every secret and configmap. On large
clusters that takes a while and shows up as API server rollouts. Don't flip it
on the whole fleet during business hours.

**FIPS hubs.** The init container copies the binary named
`PolicyGenerator-not-fips-compliant`. If your hub runs in FIPS mode, check
which build is right for you before you copy this patch.

**`--enable-alpha-plugins` is required.** Forget it and Argo CD renders the
PolicyGenerator YAML as a plain resource, then fails to apply it because
there's no such CRD. The error doesn't point you at the real cause.

## Wrapping up

Git is great for writing and reviewing config. It's a poor runtime dependency
for clusters that can't, or shouldn't, reach it. Package the policies as an
OCI artifact, let Argo CD pull them from the registry the cluster already
trusts, and let ACM handle the fleet. You get versioned, auditable baseline
config that works the same on a connected hub and an air-gapped one.

**References:**
- [Example repo: rhacm-cluster-policies](https://github.com/sumantripuraneni/rhacm-cluster-policies)
- [Argo CD docs: OCI sources](https://argo-cd.readthedocs.io/en/stable/user-guide/oci/)
- [Red Hat Developer: How to use OCI for GitOps in OpenShift](https://developers.redhat.com/articles/2025/10/13/how-use-oci-gitops-openshift)