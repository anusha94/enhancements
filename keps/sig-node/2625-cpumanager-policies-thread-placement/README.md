# cpu manager extension to reject non SMT-aligned workload

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Latency-sensitive applications runtime guarantees](#latency-sensitive-applications-runtime-guarantees)
    - [Improve the density of running containers](#improve-the-density-of-running-containers)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Implementation strategy of full-pcpus-only CPU Manager policy option](#implementation-strategy-of-full-pcpus-only-cpu-manager-policy-option)
  - [Resource Accounting](#resource-accounting)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Test Plan](#test-plan-1)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Graduation Criteria of Options](#graduation-criteria-of-options)
    - [Graduation of Options to <code>Beta-quality</code> (non-hidden)](#graduation-of-options-to-beta-quality-non-hidden)
    - [Graduation of Options from <code>Beta-quality</code> to <code>G.A-quality</code>](#graduation-of-options-from-beta-quality-to-ga-quality)
  - [Removal of the CPUManagerPolicyAlphaOptions and CPUManagerPolicyBetaOptions feature gates](#removal-of-the-cpumanagerpolicyalphaoptions-and-cpumanagerpolicybetaoptions-feature-gates)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
    - [Add extra resources](#add-extra-resources)
    - [Add a new unit for CPU resources](#add-a-new-unit-for-cpu-resources)
  - [Future extension (obsolete, recorded for the sake of history)](#future-extension-obsolete-recorded-for-the-sake-of-history)
    - [Allowing external agents to detect the cpumanager behaviour](#allowing-external-agents-to-detect-the-cpumanager-behaviour)
    - [Emulating Non-SMT on SMT systems](#emulating-non-smt-on-smt-systems)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ X (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [X] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [X] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [X] (R) Graduation criteria is in place
  - [X] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [X] (R) Production readiness review completed
- [X] (R) Production readiness review approved
- [X] "Implementation History" section is up-to-date for milestone
- [X] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [X] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

We propose a change in CPUManager to make the behaviour of latency-sensitive applications more predictable when running on SMT-enabled systems.

## Motivation

Latency-sensitive applications want to have exclusive CPU allocation to enable better isolation to meet application's latency and performance requirements.
The static policy of the CPUManager already allows to prevent virtual CPU sharing.
However, for some classes of these latency-sensitive applications running on simultaneous multithreading (SMT) enabled system, it is also beneficial
to consider thread-level allocation, to avoid physical CPU sharing and prevent possible interferences caused by noisy neighborhoods.

### Goals

* Prevent workloads from requesting cores that don't consume a full CPU by rejecting them.
  This guarantees that no physical core is shared among different containers, which improves cache efficiency and mitigates the interference
  with other workloads that can consume resources of the same physical core, e.g. first level caches.

### Non-Goals

* Add new cpumanager policies. The community feedback and the conversation we gathered when proposing this KEP were all in favor
  of adding options to fine-tune the behavior of the static policy rather than adding new policies.

## Proposal

### User Stories

#### Latency-sensitive applications runtime guarantees

Some classes of latency-sensitive applications (CNF, HFT, ML/AI) benefit of the thread placement constraints this policy enables.
More precise thread placement allows to control which physical cores are shared among containers, such as:

1. workloads may ensure the physical cores are shared among their threads for increased efficiency
2. workloads may ensure the physical cores are *not* shared with other containers for interference prevention.

An implementation of the concepts proposed here is already found in [external projects](https://github.com/nokia/CPU-Pooler#hyperthreading-support).
[OpenStack](https://specs.openstack.org/openstack/nova-specs/specs/mitaka/implemented/virt-driver-cpu-thread-pinning.html),
which is one of the leading platform for VNF (Virtual Network Functions), the predecessor of CNFs.

#### Improve the density of running containers

Thread allocation guarantees enables more efficient usage of the node resources.
The nodes can now accommodate safely mixed workloads of latency-sensitive and not-latency-sensitive (infrastructure) pods.
This increases the node usage, which reduces the need for extra hardware, which drives down the TCO of a container-based solution.

### Risks and Mitigations

This new behaviour is opt-in. Users will need to explicitly enable it in their kubelet configuration. The change is very self contained, with little impact in the shared codebase.
The impact in the shared codebase will be addressed enhancing the current testsuite.

| Risk                                                      | Impact        | Mitigation |
| --------------------------------------------------------- | ------------- | ---------- |
| Bugs in the implementation lead to kubelet crash | High | Disable the policy and restart the kubelet. The workload will run but with weaker guarantees - like it was before this change. |


## Design Details

We propose to
- add a new flag in Kubelet called `CPUManagerPolicyOptions` in the kubelet config or command line argument called `cpumanager-policy-options` which allows the user to specify the CPU Manager policy option.
- add a new cpu manager option called `full-pcpus-only`; if present, this option will enable further refinements of the existing static policy.

The static policy allocates CPUs using a topology-aware best-fit allocation. This enhancement wants to provide stronger guarantees by restricting the allocation of threads.
The aim is to achieve the isolation for workloads managed by Kubernetes. The other part of isolation is (as of now) not managed by Kubernetes, as described in [Explicitly Reserved CPU List](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#explicitly-reserved-cpu-list) and [Static policy](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy).

Let's summarize the key properties of the `full-pcpus-only` option:
- Preserve all the properties of the `static` policy.
- Never allocate less than a physical-cpu worth amount of cores.
- With this requirement enforced, the CPUManager allocation algorithm will guarantee avoidance of physical core sharing.
- Should the node not have enough free physical cores, the Pod will be put in Failed state, with `SMTAlignmentError` as reason.

### Implementation strategy of full-pcpus-only CPU Manager policy option

- In order to introduce the SMT-alignment check in CPU Manager, we introduce a new flag in Kubelet to allow the user to specify `cpumanager-policy-options` which when specified with `full-pcpus-only` as its value provides the capability to modify the behaviour of static policy to strictly guarantee allocation of whole cores to a workload.
- The `CPUManagerPolicyOptions` received from the kubelet config/command line args is propogated to the Container Manager.
- The responsibility of admission control is centralized in containermanager. The resource managers and/or the resource allocation orchestrator (Topology Manager) still have the responsibility of running the checks to admit the pods, but the handling of these errors and the building of the pod lifecycle result are now factored in containermanager.
- Prior to this feature, the Container Manager admission handler was delegated to the topology manager if the latter was enabled. This worked well under the assumption that only Topology Manager had the ability to reject admissions with pods. But with the introduction of this feature, the CPU Manager also needs the ability to possibly reject pods if strict SMT alignment is requested. In order to do so, we introduce a new error and let it drive the rejection. Due to an already existing dependency between CPUManager and TopologyManager as the former imports the latter in order to support the `topologymanager.HintProvider` interface, container manager is considered as the appropriate for performing admission control.
- When `full-pcpus-only` policy option is specified along with `static` CPU Manager policy, an additional check in the allocation logic of the `static` policy ensures that CPUs would be allocated such that full cores are allocated. Because of this check, a pod would never have to acquire single threads with the aim to fill partially-allocated cores.
- In case request translates to partial occupancy of the cores, the Pod will not be admitted and would fail with `SMTAlignmentError`.


### Resource Accounting

To illustrate the behaviour of the `full-pcpus-only` policy option, we will consider the following CPU topology. We will use as example a CPU package with 16 physical cores, 2-way SMT-capable.

![Example Topology](smtalign-topology.png)


Let's consider a single container, requesting 5 exclusive cores.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "128Mi"
        cpu: "5"
      limits:
        memory: "128Mi"
        cpu: "5"
```

The `full-pcpus-only` policy option will cause the pod to be rejected since it doesn't request enough cores to consume all virtual threads exposed by the CPU.

 would need to make sure the remaining core on the half-allocated physical CPU is left unallocated to avoid noisy neighbours.

![Example core allocation with the full-pcpus-only policy option when requesting a odd number of cores](smtalign-allocation-odd-cores.png)

The container will then actually get more virtual cores (6) than what is requesting (5).

| Requested (virtual) CPUs | Requested (physical) CPUs | Unallocatable (virtual) CPUs   | Total allocated (virtual) CPUs            |
| ------------------------ | ------------------------- | ------------------------------ | ----------------------------------------- |
| 5                        | 3                         | 1                              | 6                                         |
| N                        | M = N / `threads_per_cpu` | X = N % `threads_per_cpu`      | `requested_vcpus` + `unallocatable_vcpus` |

With `threads_per_cpu` is typical 2 on x86_64 with SMT enabled - but this number is not fixed and can change in future hardware implementation.

In order to make the resource reporting consistent, and avoiding cascading changes in the system, we enforce the request constraints at admission time.
This approach follows what the Topology Manager already does.

### Test Plan

[ ] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

- `k8s.io/kubernetes/pkg/kubelet/cm/cpumanager`:            `20250130` - 85.6% of statements
- `k8s.io/kubernetes/pkg/kubelet/cm/cpumanager/state`:      `20250130` - 88.1% of statements
- `k8s.io/kubernetes/pkg/kubelet/cm/cpumanager/topology`:   `20250130` - 85.0% of statements

##### Integration tests

- kubelet features don't have usually integration tests. We use a combination of unit tests and `e2e_node` tests.

##### e2e tests

testgrid:
- https://testgrid.k8s.io/sig-node-kubelet#kubelet-serial-gce-e2e-cpu-manager
- https://testgrid.k8s.io/sig-node-kubelet#kubelet-gce-e2e-arm64-ubuntu-serial
- https://testgrid.k8s.io/sig-node-containerd#pull-e2e-serial-ec2
- https://testgrid.k8s.io/sig-node-containerd#node-kubelet-containerd-resource-managers

### Test Plan

The [implementation PR](https://github.com/kubernetes/kubernetes/pull/101432) will extend both the unit test suite and the E2E test suite to cover the policy changes described in this KEP.

### Graduation Criteria

#### Alpha
- [X] Implement the new policy.
- [X] Ensure proper e2e node tests are in place.

#### Beta
- [X] Gather feedback from the consumer of the policy.
- [X] No major bugs reported in the previous cycle.
- [X] Use of this policy option to further configure the behavior of CPU manager. Another CPUManager policy option `distribute-cpus-across-numa` is being proposed in 1.23 release to distribute CPUs across NUMA nodes instead of packing them.

#### GA
- [X] Allowing time for feedback (1 year).
- [X] Risks have been addressed.

### Graduation Criteria of Options

In 1.23 release, as we are graduating this feature to Beta meaning `CPUManagerPolicyOptions` is enabled by default allowing the user to configure CPU Manager static policy with the option `full-pcpus-only`.
NOTE: Even though the feature gate is enabled by default the user still has to explicitly use the Kubelet flag called `CPUManagerPolicyOptions` in the kubelet config or command line argument called `cpumanager-policy-options` along with a specific policy option to use this feature.
- In addition to this, in order to not have all alpha-quality experimental options introduced in the future available by default, we are introducing an additional feature gates called `CPUManagerPolicyAlphaOptions` and `CPUManagerPolicyBetaOptions`.
  These feature gates control all the non-stable options. We introduce feature gates guarding groups of options to avoid the proliferation of feature gates, and to make the management easier: tracking a feature gate per option would be too cumbersome.
  The alpha-quality options are hidden by default and only if the `CPUManagerPolicyAlphaOptions` feature gate is enabled the user has the ability to use them.
  The beta-quality options are visible by default, and the feature gate allows a positive acknowledgement that non stable features are being used, and also allows to optionally turn them off.
  Based on the graduation criteria described below, a policy option will graduate from a group to the other (alpha to beta).
- Since the feature that allows the ability to customize the behaviour of CPUManager static policy as well as the CPUManager Policy option `full-pcpus-only` were both introduced in 1.22 release and meet the above graduation criterion, `full-pcpus-only` would be considered as a non-hidden option i.e. available to be used when explicitly used along with `CPUManagerPolicyOptions` Kubelet flag in the kubelet configuration or command line argument called `cpumanager-policy-options` .
-  The introduction of this new feature gate gives us the ability to move the feature to beta and later stable without implying all that the options are beta or stable.

The graduation Criteria of options is described below:

#### Graduation of Options to `Beta-quality` (non-hidden)
- [X] Gather feedback from the consumer of the policy option.
- [X] No major bugs reported in the previous cycle.

#### Graduation of Options from `Beta-quality` to `G.A-quality`
- [X] Allowing time for feedback (1 year) on the policy option.
- [X] Risks have been addressed.

### Removal of the CPUManagerPolicyAlphaOptions and CPUManagerPolicyBetaOptions feature gates

This KEP added the `CPUManagerPolicyAlphaOptions` and `CPUManagerPolicyBetaOptions` group feature gates alongside the usual changes required to enable the `full-pcpus-only` option.

We plan to remove the `CPUManagerPolicyAlphaOptions` and `CPUManagerPolicyBetaOptions` after all options graduated to stable.
We will defer to the last graduating option the additional work to remove the gates. In case of two or more options graduating to GA and thus rendering the gates obsolete, a new minimal KEP should be issued to remove the gates.

The SIG-node community is considering a redesign of the resource management based on NRI and DRA technologies (possibly extended) for the future.
There were conversation and attempts about this topic since the 1.27 cycle and [past attempts already](https://github.com/kubernetes/enhancements/issues/3675).
We thus expect a gradual slowdown of additions to cpumanager, including policy options. At time of writing (1.33 cycle) we have 6 policy options at various degree of maturity.
We expect the rate of proposal of new options to greatly slow down and to stop entirely once the community moves to the future, yet unplanned, resource management architecture.

For the reasons above, we believe it's unlikely we will need to add back the `CPUManagerPolicyAlphaOptions` and `CPUManagerPolicyBetaOptions` feature gates once removed.
Should new options be proposed and agreed by the community, the recommendation is to graduate using specific feature gates per standard process.
Considering the expected slowdown, we expect the standard graduation process to be much more manageable in the future, if needed at all.

### Upgrade / Downgrade Strategy

We expect no impact. The new policies are opt-in and separated by the existing ones.

### Version Skew Strategy

No changes needed

## Production Readiness Review Questionnaire

### Feature enablement and rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: `CPUManagerPolicyOptions` - going to be locked to true once GA.
    - Feature gate name: `CPUManagerPolicyBetaOptions`.
    - Feature gate name: `CPUManagerPolicyAlphaOptions`.
    - Components depending on the feature gate: kubelet
- [X] Other
  - Change the kubelet configuration adding the CPUManager policy option to `full-pcpus-only`
  - Will enabling / disabling the feature require downtime of the control
    plane? No
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node? No

###### Does enabling the feature change any default behavior?
  - Enabling the `CPUManagerPolicyOptions` makes the behaviour of the CPUManager static policy more restrictive and can lead to pod admission rejection.
  - Enabling the `CPUManagerPolicyAlphaOptions` provides the ability to use alpha-quality options which can change the behaviour of the CPUManager static policy.
  - Enabling the `CPUManagerPolicyBetaOptions` provides the ability to use beta-quality options which can change the behaviour of the CPUManager static policy.
    This allows a positive acknowledgment from the cluster admin that they are using a non-stable feature; additionally, the feature gate allows to _disable_ a set of options.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

  - Yes, disabling the `CPUManagerPolicyOptions` feature gate shuts down the feature completely; alternatively through kubelet configuration - switch to a different policy.
  - Also, disabling `CPUManagerPolicyAlphaOptions` or `CPUManagerPolicyBetaOptions` feature gates disables the use of non-stable options, respectively alpha or beta quality,
    and the behavior would depend on how `CPUManagerPolicyOptions` feature gate is configured.
  - Disabling both the feature gates would allow complete rollback of this enablement.

###### What happens if we reenable the feature if it was previously rolled back?

No changes. Existing containers will not see their allocation changed. New containers will.

###### Are there any tests for feature enablement/disablement?

A specific e2e test will demonstrate that the default behaviour is preserved when the `CPUManagerPolicyOptions` feature gate is disabled, or when the feature is not used (2 separate tests)

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

Kubelet may fail to start. The kubelet may crash.

###### What specific metrics should inform a rollback?

We can use `cpu_manager_pinning_errors_total` to see all the allocation errors, irrespective of the specific reason though.
In addition, we can use the logs: the number of pod ending up in Failed for SMTAlignmentError could be used to decide a rollback.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Not Applicable.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring requirements

###### How can an operator determine if the feature is in use by workloads?

- Check the metric `container_aligned_compute_resources_count` with the label `boundary=physical_cpu`
- Inspect the kubelet configuration of the nodes: check feature gates and usage of the new options

###### How can someone using this feature know that it is working for their instance?

- [X] Other (treat as last resort)
  - Details:
    - check metrics and their interplay:
        * the metric `container_aligned_compute_resources_count` with the label `boundary=physical_cpu`
        * the metric `cpu_manager_pinning_requests_total`
        * the metric `cpu_manager_pinning_errors_total`

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

N/A.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [X] Metrics
  - Metric name:
    * the metric `container_aligned_compute_resources_count` with the label `boundary=physical_cpu`
        * the metric `cpu_manager_pinning_requests_total`
        * the metric `cpu_manager_pinning_errors_total`
  - Components exposing the metric: kubelet

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

We can detail the pinning errors total with a new metric like `cpu_manager_errors_count` or
`container_aligned_compute_resources_failure_count` using the same labels as we use for `container_aligned_compute_resources_count`.
These metrics will be added before to graduate to GA.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

No effect.

###### What are other known failure modes?

Allocation failures can lead to workload not going running. The only remediation is to disable the features and restart the kubelets.

###### What steps should be taken if SLOs are not being met to determine the problem?

Inspect the metrics and possibly the logs to learn the failure reason

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

- 2021-04-14: KEP created
- 2021-04-16: KEP updated with the `smtisolate` policy
- 2021-04-19: KEP updated to capture implementation details of the `smtaware` policy; clarified the resource accounting vs admission requirements
- 2021-04-22: KEP updated to clarify the `smtaware` policy after discussion on sig-node and to postpone the `smtisolate` policy
- 2021-05-04: KEP updated to change name from `smtaware` to `smtalign`. In addition to this we capture changes in the implmentation details including the introduction of a new flag in Kubelet called `cpumanager-policy-options` to allow the user to specify `smtalign` as a value to enable this capability.
- 2021-05-06: KEP update to add the feature gate and clarify PRR answers.
- 2021-05-10: KEP update to add to rename the `smtalign` to `reject-non-smt-aligned` for better clarity and address review comments
- 2021-05-11: KEP update to add to the `configurable` alias and address review comments
- 2021-05-13: KEP update to postpone the `configurable` alias, per review comments
- 2021-09-02: KEP update to capture the policy name `full-pcpus-only` based on the implementation merged in 1.22, explain how this feature is being used for introduction of another policy option and updates pertaining to promotion of the feature to Beta.
- 2021-09-08: KEP update to introduce `CPUManagerPolicyExperimentalOptions` feature gate to prevent alpha-quality options from being non-hidden (available) by default and explain the graduation criteria of the options.
- 2021-09-27: KEP update to split `CPUManagerPolicyExperimentalOptions` feature gate into `CPUManagerPolicyAlphaOptions` and `CPUManagerPolicyBetaOptions` after discussion on [sig-arch](https://groups.google.com/u/1/g/kubernetes-sig-architecture/c/Nxsc7pfe5rw)
- 2025-01-30: KEP update to align with most recent template and add changes for GA graduation

## Alternatives

We acknowledge few drawbacks of the proposed approach:
- pods that are rejected due to an AdmissionError do not get automatically rescheduled. Workloads which wants to make sure to be rescheduled need to
  use extra kubernetes facilities, for example Deployments.
- pods might have to overallocate resources.

We evaluated possible alternatives to the extra admission control, but we eventually discarded all of them. We document them in this section.

#### Add extra resources

We can add a new extended resource alongside `cpu` - [which on baremetal represents virtual threads](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/#cpu-units), to represent
physical CPUs. However having two resources to represent the same hardware entity is confusing and cumbersome. We believe CPUManager should keep consuming the core `cpu` resource for consistency reasons.

Just considering the new extended resource, is not feasible as well, because it will prevent the pod to be in the guaranteed QoS class, and will void the desirable property of keeping all the guarantees
the static policy provides.

#### Add a new unit for CPU resources

Since a physical core always hosts one or more virtual thread, [hence one or more CPUs](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/#cpu-units), we can
add a resource qualifier to represent such multiple. For example, the `p` qualifier in the example below could allow the users to change the meaning of the value such as the request
is expressed in terms of physical CPUs:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "128Mi"
        cpu: "3p"
      limits:
        memory: "128Mi"
        cpu: "3p"
```

This approach however is relevant only when kubernetes runs on baremetal machines, and irrelevant on the cloud; More over it will make the definition more ambiguous,
because the relationship between physical CPUs and virtual CPUs is hardware and configuration dependent:

| Environment                                          | Virtual/Physical cpu ratio |
| ---------------------------------------------------- | -------------------------- |
| Baremetal, 2-way SMT capable (x86_64), SMT enabled   | 2                          |
| Baremetal, 2-way SMT capable, SMT disabled           | 1                          |

SMT can usually be disabled via software (kernel parameter, firmware settings), and [other implementations than 2-way SMT are possible](https://en.wikipedia.org/wiki/Simultaneous_multithreading).
Hence, the ratio between virtual and physical CPUs is not fixed and is not predictable, subject to changes in future.
Furthermore, this new unit would make sense only for CPU resources, and not for all the other resources.


### Future extension (obsolete, recorded for the sake of history)

#### Allowing external agents to detect the cpumanager behaviour

As soon as a behaviour-changing option is added, we should add an alias to `static`, which we proposed to name `configurable`.

This is motivated by the fact some operators may make logic on the reported CPUManager policy, and not notice the extra flags that potentially change the behaviour of the static policy.
An example of these aforementioned operators are software components who vendored older kubelet configuration API (predating this work).
By adding the new policy alias, we make it possible for those operators to trivially detect the change, and to not make wrong assumptions.

#### Emulating Non-SMT on SMT systems

We would like to mention a further extension of this work, which we are *not* proposing here.

A further subset of the latency sensitive class of workload we identified (CNF, HFT) benefits most of non-SMT system, delivering the best possible performance here.
For these applications, just disabling SMT at machine level solves the need of the workload, but overall creates worse usage of hardware resources and poorer container density.

Another policy option, or a further refinement of `full-pcpus-only`, which enables non-SMT emulation on SMT-enabled system would allow to accommodate these needs, but this would cause even more significant resource accounting mismatches
as described above. Furthermore, at the moment of writing we are still assessing how large is the set of the classes which benefit of these extra guarantees.

For all these reasons we postponed this work to a later date.
