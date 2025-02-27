# KEP-784: Kube Proxy component configuration graduation

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This document is intended to propose a process and desired goals by which kube-proxy's component configuration is to be graduated to beta.

## Motivation

kube-proxy is a component, that is present in almost all Kubernetes clusters in existence.
Historically speaking, kube-proxy's configuration was supplied by a set of command line flags. Over time, the number of flags grew, and they became unwieldy to use and support. Thus, kube-proxy gained component config.
Initially this was just a large flat object, that was representing the command line flags. However, over time new features were added to it, all while staying as v1alpha1.

This resulted in a configuration format, that had various different options grouped together in ways, that made them hard to specify and understand. For example:

- Instance local options (such as hostnameOverride, bindAddress, etc.) are in the same flat object as shared between instances options (such as clusterCIDR, configSyncPeriod, etc.).
- Platform specific options are marked as generic options (eg. conntrack, oomScoreAdj).
- Backend agnostic options are marked as backed specific options (eg. syncPeriod,  minSyncPeriod).
- Options specific to a backend are used by other backends (eg. masqueradeBit and masqueradeAll).

[kubernetes/issues/117909](https://github.com/kubernetes/kubernetes/issues/117909) captures all the mis configurations in details.

Clearly, this made the configuration both hard to use and to maintain. Therefore, a plan to restructure and stabilize the config format is needed.

### Goals

- To clean up the existing config format.
- To provide config structure, that is easier for users to understand and use.
- To distinguish between instance local and shared settings.
- To allow for the persistence of settings for different platforms (such as Linux and Windows) in a manner that reduces confusion and the possibility of an error.
- To allow for easier introduction of new proxy backends.
- To provide users with flexibility, especially in regard to the config source.

### Non-Goals

- To change or implement additional features in kube-proxy.
- To deal with graduation of any other component of kube-proxy, other than its configuration.
- To remove most or even all the command line flags, that have corresponding component config options.

## Proposal

The idea is to conduct the process of graduation to beta in small steps in the span of at least one Kubernetes release cycle.
This will be done by creating one or more alpha versions of the config with the last alpha version being copied as v1beta1 after
the community is happy with it. Each of the subsections below can result in a separate alpha version release, although it will
be better for users to have no more than a couple of alpha versions past v1alpha1. After each alpha version release, the community
will gather around for new ideas on how to proceed in the graduation process. If there are viable proposals, this document is
updated with an appropriate section(s) below and the new changes are introduced in the form of new alpha version(s). The proposed
process is similar to the already successfully used one for kubeadm.

The current state of the config has proven that:
- Some options are deemed as mode specific, but are in fact shared between all modes.
- Some options are placed directly into KubeProxyConfiguration, but are in fact mode specific ones.
- There are options that are shared between some (but not all) modes. Specific features of the underlying implementation are common and this happens only within the boundaries of the platform (iptables and ipvs modes for example).

With that in mind, the following measures are proposed:
- Create platform subsection for platform specific fields.
- Move backend-agnostic and platform-agnostic fields from backend section to root section.
- Move backend-agnostic and platform-specific fields from backend section to relevant platform section.
- Drop legacy/unused options. 

### Risks and Mitigations

So far, the following risks have been identified:
- Deviation of the implementation guidelines and bad planning may have the undesired effect of producing bad alpha versions.
- Bad alpha versions will need good alpha versions to fix them. This will create too many iterations over the API and users may get confused.
- New and redesigned kube-proxy API versions may cause confusion among users who are used to the v1alpha1 relatively flat, single document design.
  In particular, multiple YAML documents and structured (as opposed to flat) objects can create confusion as to what option is placed where.

The mitigations to those risks:
- Strict following of the proposals in this document and planning ahead for a release and config cycle.
- Support reading from the last couple of API versions released. When the beta version is released, support the last alpha version for one or two release cycles after that.
- Documentation on the new APIs and how to migrate to them.
- Provide optional migration tool for the APIs.

## Design Details


### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates


##### Unit tests


##### Integration tests


##### e2e tests

### Graduation Criteria

The config should be considered graduated to beta if it:
- is well-structured with clear boundaries between different proxy mode settings.
- allows for easy multi-platform use with less probability of an error.
- allows for easy distinction between instance local and shared settings.
- is well covered by tests.
- is well documented. Especially in regard to migrating to it from older versions.

### Upgrade / Downgrade Strategy


### Version Skew Strategy


## Production Readiness Review Questionnaire


### Feature Enablement and Rollback


###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [ ] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name:
  - Components depending on the feature gate:
- [x] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->

###### What happens if we reenable the feature if it was previously rolled back?

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

###### What specific metrics should inform a rollback?

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

###### How can someone using this feature know that it is working for their instance?

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

No.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. 

### Scalability


###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

Yes.
[WIP]

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes.
[WIP]

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

### Troubleshooting


###### How does this feature react if the API server and/or etcd is unavailable?


###### What are other known failure modes?


###### What steps should be taken if SLOs are not being met to determine the problem?


## Implementation History


## Drawbacks

## Alternatives

## Infrastructure Needed (Optional)

