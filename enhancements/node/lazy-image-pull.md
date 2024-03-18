---
title: lazy-image-pull
authors:
  - @harche
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - @mrunalp
  - @rphillips
approvers: # A single approver is preferred, the role of the approver is to raise important questions, help ensure the enhancement receives reviews from all applicable areas/SMEs, and determine when consensus is achieved such that the EP can move forward to implementation.  Having multiple approvers makes it difficult to determine who is responsible for the actual approval.
  - @mrunalp
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None"
  - @mrunalp
creation-date: 2024-03-18
last-updated: 2024-03-18
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
  - https://issues.redhat.com/browse/OCPNODE-2204
---

# Enable Lazy Image Pulling Support in OpenShift


## Summary

This enhancement proposes integrating support for lazy image pulling in OpenShift using [Stargz Store plugin](https://github.com/containerd/stargz-snapshotter). This will provide the ability to lazy-pull compatible container images, significantly reducing container startup times and potentially decreasing network strain for the specific workloads that can benefit from it.

## Motivation

* **Faster Serverless:** - Serverless end points need to serve the cold requests as fast as possible. Lazy-pulling of the images allow the serverlesss end point to start serving the user request without getting blocked on the entire container image to get downloaded on the host.
* **Improved User Experience:** Large images often lead to extended pod startup times. Lazy-pulling drastically reduces the initial delay, providing a smoother user experience.
* **Potential Benefits for Large Images:** If the container image is shared between various containers which utilize only part of the image during their execution, Lazy image pulling speeds up such container startup by downloading only the relavent bits from those images.

### User Stories


* As a cluster administrator, I want to serve serverless endpoints on my cluster as fast as possible by not waiting for the entire container image to download.
* As a developer, I want to deploy applications packaged in large container images while minimizing the impact of image size on pod startup time.
* As a cluster administrator, I want to reduce network bandwidth usage in my OpenShift environment, especially when dealing with large container images.

# Goals

* Modify `c/storage` to support drop-in config
* Package upstream [Stargz Store plugin](https://github.com/containerd/stargz-snapshotter) for the installation on RHCOS
* Enable lazy image pulling in Openshift using [Stargz Store plugin](https://github.com/containerd/stargz-snapshotter) which uses [eStargz](https://github.com/containerd/stargz-snapshotter/blob/main/docs/estargz.md) image format.
* Support lazy image pulling only on the worker nodes (or subset of worker nodes).

# Non-Goals

* Enabling lazy image pulling on the control plane nodes.
* Replacement of the standard OCI image format; `eStargz` will be an additional supported format.
* Alterations to the image build process; developers can use standard OCI build tools, such as buildah to build and convert images to `eStargz` format.

## Proposal

In order to use `eStargz` images, we need to have [Stargz Store plugin](https://github.com/containerd/stargz-snapshotter/blob/aaa46a75dd97e401025f82630c9d3d4e41c9f670/docs/INSTALL.md#install-stargz-store-for-cri-opodman-with-systemd) installed on the node and with a configuration for container storage library. Existing MCO [node controller](https://github.com/openshift/machine-config-operator/blob/53bbe70847393935e8f2c3f83c739fbb477cb84d/pkg/controller/node/node_controller.go#L75) for `node.config` object can be used to roll out this configuration to all or [set of specific worker nodes](https://github.com/openshift/machine-config-operator/blob/release-4.16/docs/custom-pools.md).

### Workflow Description

* A cluster administrator will create or edit the [node.config](https://github.com/openshift/api/blob/610cbc77dbab208d5364bba8982385bd148025c1/config/v1/types_node.go#L19) object to enable lazy image pulling support for all worker nodes by, `oc edit nodes.config -o yaml`
```yaml
apiVersion: config.openshift.io/v1
kind: Node
metadata:
  name: cluster
  spec:
    lazyImagePools:
      - "worker"
```
or for specific machine config pool(s) by,
```yaml
apiVersion: config.openshift.io/v1
kind: Node
metadata:
  name: cluster
  spec:
    lazyImagePools:
      - "serverless-custom-pool1"
      - "serverless-custom-pool2"
```
*  Existing MCO [node controller](https://github.com/openshift/machine-config-operator/blob/53bbe70847393935e8f2c3f83c739fbb477cb84d/pkg/controller/node/node_controller.go#L75) for `node.config` object, which is responsible for reconciling any changes to that object, would recongize the user's intent to enable the support for lazy image pulling.
* Before rolling out the configuration to the specified machine config pools, node controller will validate that they are not inherited from the `master` pool.
* Node controller then provisions a machine config consisting of required changes that would be rolled out to specified machine config pool(s).

### API Extensions

Extend existing [NodeSpec](https://github.com/openshift/api/blob/610cbc77dbab208d5364bba8982385bd148025c1/config/v1/types_node.go#L36) to hold the user specified machine config pool(s).
```golang=
type NodeSpec struct {
    LazyImagePools []string `json:"lazyImagePools,omitempty"`
}
```

### Topology Considerations

#### Hypershift / Hosted Control Planes

N/A

#### Standalone Clusters

N/A

#### Single-node Deployments or MicroShift

N/A

### Implementation Details/Notes/Constraints

## Package Stargz Store plugin

Package the Stargz Store plugin located at, https://github.com/containerd/stargz-snapshotter so that it can get installed on RHCOS

## Drop-in config support for c/storage
A configuration for `additionallayerstores` needs to be added in `/etc/containers/storage.conf.d/estargz.conf` on the node as follows,
```toml=
[storage.options]
additionallayerstores = ["/var/lib/stargz-store/store:ref"]
```
As of now, that `c/storage` configuration file does not support drop-in config. The entire content of that file is [hard-coded in MCO](https://github.com/openshift/machine-config-operator/blob/release-4.16/templates/common/_base/files/container-storage.yaml). Without the drop-in config support, we risk overriding that critical file and potentially losing custom configuration that might be already present in there. A Request for adding drop-in config support for applying similar custom configurations to `c/storage` [already exists](https://github.com/containers/storage/issues/1306). Hence, this proposal advocates adding support for drop-in config in `c/storage`.



## Starz Store systemd service

Startz Store service needs to be running on the host _before_ crio service starts. MCO needs to drop relavent systemd unit file on the targeted nodes.

```toml=
[Unit]
Description=Stargz Store plugin
After=network.target
Before=crio.service

[Service]
Type=notify
Environment=HOME=/home/core
ExecStart=/usr/local/bin/stargz-store --log-level=debug --config=/etc/stargz-store/config.toml /var/lib/stargz-store/store
ExecStopPost=umount /var/lib/stargz-store/store
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

### Risks and Mitigations

## Enhancing c/storage Configuration Management

Even in the absence of drop-in configuration support within `c/storage`, the current [hard-coded](https://github.com/openshift/machine-config-operator/blob/release-4.16/templates/common/_base/files/container-storage.yaml) `/etc/containers/storage.conf` file can be updated by the MCO. However, the integration of drop-in configuration support would offer a more secure and robust method for managing this configuration file. Should the drop-in configuration feature not be implemented in `c/storage`, the MCO has the capability to read the existing, hard-coded `storage.conf` and appropriately inject the necessary `additionallayerstores` settings.

### Drawbacks

#### Resource Consumption with FUSE

The stargz-snapshotter employs a FUSE filesystem, which can lead to increased resource usage on the node during intensive local disk I/O operations. It’s important to note that this does not affect containers performing heavy I/O with local volume mounts, as they will not experience performance degradation.


## Test Plan

#### OpenShift End-to-End Testing

Write comprehensive e2e tests to confirm the accurate pulling and execution of containers utilizing eStargz images.

#### Performance Benchmarking

Benchmarks comparing the performance of eStargz images against traditional image usage across diverse operational scenarios.


## Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
  - [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
  - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**If this is a user facing change requiring new or updated documentation in [openshift-docs](https://github.com/openshift/openshift-docs/),
please be sure to include in the graduation criteria.**

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels].

### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

## Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
  disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
  this should be identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

## Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Operational Aspects of API Extensions

Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
especially how they impact the OCP system architecture and operational aspects.

- For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
  Indicators) an administrator or support can use to determine the health of the API extensions

  Examples (metrics, alerts, operator conditions)
  - authentication-operator condition `APIServerDegraded=False`
  - authentication-operator condition `APIServerAvailable=True`
  - openshift-authentication/oauth-apiserver deployment and pods health

- What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
  API availability)

  Examples:
  - Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
  - Fails creation of ConfigMap in the system when the webhook is not available.
  - Adds a dependency on the SDN service network for all resources, risking API availability in case
    of SDN issues.
  - Expected use-cases require less than 1000 instances of the CRD, not impacting
    general API throughput.

- How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
  automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
  this enhancement)

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
  (e.g. which kube-controller-manager functionality will stop working), especially regarding
  stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
  and add them as reviewers to this enhancement.

## Support Procedures

Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
  alerts, which log output in which component)

  Examples:
  - If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
  - Operator X will degrade with message "Failed to launch webhook server" and reason "WehhookServerFailed".
  - The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
    will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

  - What consequences does it have on the cluster health?

    Examples:
    - Garbage collection in kube-controller-manager will stop working.
    - Quota will be wrongly computed.
    - Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
      Disabling the conversion webhook will break garbage collection.

  - What consequences does it have on existing, running workloads?

    Examples:
    - New namespaces won't get the finalizer "xyz" and hence might leak resource X
      when deleted.
    - SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
      communication after some minutes.

  - What consequences does it have for newly created workloads?

    Examples:
    - New pods in namespace with Istio support will not get sidecars injected, breaking
      their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
  consistency?

  Examples:
  - The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
    will not block the creation or updates on objects when it fails. When the
    webhook comes back online, there is a controller reconciling all objects, applying
    labels that were not applied during admission webhook downtime.
  - Namespaces deletion will not delete all objects in etcd, leading to zombie
    objects when another namespace with the same name is created.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used
to highlight and record other possible approaches to delivering the
value proposed by an enhancement, including especially information
about why the alternative was not selected.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.