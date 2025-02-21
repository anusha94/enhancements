# KEP-2258: Node log query

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Implement client for logs endpoint (OS agnostic)](#implement-client-for-logs-endpoint-os-agnostic)
  - [Linux distributions with systemd / journald](#linux-distributions-with-systemd--journald)
  - [Linux distributions without systemd / journald](#linux-distributions-without-systemd--journald)
  - [Windows](#windows)
  - [User Stories](#user-stories)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Large log files and events](#large-log-files-and-events)
    - [Wider access to all node level service logs](#wider-access-to-all-node-level-service-logs)
- [Design Details](#design-details)
    - [kubelet](#kubelet)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
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
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [x] (R) Graduation criteria is in place
- [x] (R) Production readiness review completed
- [x] (R) Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
- [x] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [x] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

A Kubernetes cluster administrator has to log in to the relavant control-plane
or worker nodes to view the logs of the API server, kubelet etc. Or they would
have to implement a client side reader. A simpler and more elegant method would
be to allow them to use a kubelet API or kubectl plugin to also view these logs
similar to using it for other interactions with the cluster. Given the sensitive
nature of the information in node logs, this feature will only be available to
cluster administrators.

## Motivation

Troubleshooting issues with control-plane and worker nodes typically requires
a cluster administrator to SSH into the nodes for debugging. While certain
issues will require being on the node, issues with the kube-proxy or kubelet,
to name a couple, could be solved by perusing their logs. However this
too requires the administrator to SSH access into the nodes. Having a way for
them to view the logs using a kubelet API or kubectl plugin will significantly
simplify their troubleshooting.


### Goals
Provide a cluster administrator with a streaming view of logs using a kubelet
API without them having to implement a client side reader or logging into the
node. This would work for:
- Services on Linux worker and control plane nodes:
  - That have systemd / journald support.
  - That have services that log to `/var/log/`
- Windows worker nodes (all supported variants) that log to `C:\var\log`,
  and Application logs.

### Non-Goals
- Providing support for non-systemd Linux distributions.
- Reporting logs for nodes that have config or connection issues with the
  cluster.
- Getting logs from services that do not use /var/log/.

## Proposal

### Implement client for logs endpoint (OS agnostic)
- Implement a client for the `/proxy/logs/` kubelet endpoint viewer.

### Linux distributions with systemd / journald
Supplement the the `/proxy/logs/` endpoint viewer on the kubelet with a thin shim
over the `journal` directory that shells out to journalctl.

### Linux distributions without systemd / journald
Running the new "kubectl node-logs" command against services on nodes that do
not use systemd / journald should return "OS not supported". However getting
logs from `/var/log/` should work on all systems.

### Windows
Reuse the kubelet API for querying the Linux journal for invoking the
`Get-WinEvent` cmdlet in a PowerShell.

### User Stories

Consider a scenario where pods / containers are refusing to come up on certain
nodes. As mentioned in the motivation section, troubleshooting this scenario
involves the cluster administrator to SSH into nodes to scan the logs. Allowing
them to use the kubelet API to do the same as they would to debug issues with a
pod / container would greatly simply their debug workflow. This also opens up
opportunities for tooling and simplifying automated log gathering. The feature
can also be used to debug issues with Kubernetes services especially in Windows
nodes that run as native Windows services and not as DaemonSets or Deployments.

Here are some example of how a cluster administrator would use this feature:
```
# Fetch kubelet logs from a node named node-1.example
kubectl get --raw "/api/v1/nodes/node-1.example/proxy/logs/?query=kubelet"

# Fetch kubelet logs from a node named node-1.example that have the word "error"
kubectl get --raw "/api/v1/nodes/node-1.example/proxy/logs/?query=kubelet&pattern=error"

# Display foo.log from a node name node-1.example
kubectl get --raw "/api/v1/nodes/node-1.example/proxy/logs/?query=/foo.log
```

### Risks and Mitigations

#### Large log files and events
If the log that is attempted to be viewed is very large (GBs) there is
potential for the node performance to be degraded. To mitigate this we only
allow returning of messages that can be retrieved within 30 seconds.

#### Wider access to all node level service logs
The cluster administrator can now view all logs in /var/log/, systemd/journald
services and Windows services. Given that the cluster administrator can log
into the nodes and view the same information this should not be an issue.
However there is potential for scenarios where the cluster administrator does
not have access to the infrastructure. This again would benefit from real world
usage feedback.

## Design Details

#### kubelet

The kubelet already has a `/var/log/` [endpoint viewer](https://github.com/kubernetes/kubernetes/blob/b184272e278571d1e6650605dd4c39be897eaaa2/pkg/kubelet/kubelet.go#L1403)
that is lacking a client. Given its existence we can supplement that with a
wafer thin shim that shells out to journalctl. This allows us to extend the
endpoint for getting logs from the system journal on Linux systems that support
systemd. To enable filtering of logs, we can reuse the existing filters
supported by journalctl.

On the Windows side viewing of logs from services that use `C:\var\log` will
be supported by the existing endpoint. For Windows services that log to the
the Application logs,we can leverage the
[Get-WinEvent cmdlet](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-7.1)
that supports getting logs from all these sources. The cmdlet has filtering
options that can be leveraged to filter the logs in the same manner we do
with the journal logs.

Please note that filtering will not be available for logs in `/var/log/` or
`C:\var\log\`.

The complete list of options that can be used are:

Option | Description
------ | -----------
`boot` | boot show messages from a specific system boot
`pattern` | pattern filters log entries by the provided PERL-compatible regular expression
`query` | query specifies services(s) or files from which to return logs (required)
`sinceTime` | an [RFC3339](https://www.rfc-editor.org/rfc/rfc3339) timestamp from which to show logs (inclusive)
`untilTime` | an [RFC3339](https://www.rfc-editor.org/rfc/rfc3339) timestamp until which to show logs (inclusive)
`tailLines` | specify how many lines from the end of the log to retrieve; the default is to fetch the whole log

The feature now enables the cluster administrator to interrogate all services.
This could be prevented by having a whitelist of allowed services. But this
comes with severe disadvantages as there could be nodes (especially with
Windows) that have other services to support networking and monitoring.
These services are variable and will depend on how the nodes have been
configured. Here are some examples:
- [hybrid-overlay-node](https://github.com/ovn-org/ovn-kubernetes/tree/master/go-controller/hybrid-overlay)
- [windows-exporter](https://github.com/prometheus-community/windows_exporter).


The `/var/log/` endpoint is enabled using the `enableSystemLogHandler` kubelet
configuration options. To gain access to this new feature, this option and a
newly introduced `enableSystemLogQuery` needs to be enabled. In addition when
introducing this feature it will be hidden behind a `NodeLogQuery` feature gate
in the kubelet that needs to be explicitly enabled. So you need to enable both
options to get access to this new feature. Disabling `enableSystemLogQuery`
will disable the new feature irrespective of the `NodeLogQuery` feature gate.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

Add unit tests to kubelet and kubectl that exercise the new arguments that
have been added.

Given that a new kubelet package is introduced as part of this feature there is
no existing test coverage to link to.

##### Integration tests

Given that we need the kubelet running locally to test this feature, integration
tests will not be possible for this feature.

##### e2e tests

Tests have been added that query the kubelet service logs on Linux nodes and
Microsoft-Windows-Security-SPP logs on Windows nodes with various options.

These tests are part of the [kubelet node](https://github.com/kubernetes/kubernetes/blob/master/test/e2e/node/kubelet.go)
e2e tests that are run as a daily periodic job:
- https://testgrid.k8s.io/sig-windows-master-release#capz-master-windows-alpha-nodelogquery

This job runs tests against both Windows and Linux nodes.

### Graduation Criteria

The plan is to introduce the feature as alpha in the v1.27 time frame behind the
`NodeLogQuery` kubelet feature gate and `enableSystemLogQuery` kubelet option.

#### Alpha -> Beta Graduation

The plan is to graduate the feature to beta in the v1.30 time frame. So far we
have not received any negative feedback from cluster administrators and
developers who have enabled the feature.

A [kubectl plugin](https://github.com/aravindhp/kubectl-node-logs) has been released
and added to the Krew [index](https://github.com/kubernetes-sigs/krew-index/blob/master/plugins/node-logs.yaml)
for querying the logs more elegantly instead of using raw API calls.

#### Beta -> GA Graduation

The plan is to graduate the feature to GA in the v1.32 time frame at which point
any major issues should have been surfaced and addressed during the alpha and
beta phases.

### Upgrade / Downgrade Strategy

### Version Skew Strategy

If the API call is made against a kubelet that does not support the new feature,
a 404 will be returned.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [x] Feature gate
    - Feature gate name: NodeLogQuery
    - Components depending on the feature gate: kubelet

* **Does enabling the feature change any default behavior?** No

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?** Yes. It can be disabled by disabling the `NodeLogQuery` feature
  gate in the kubelet.

* **What happens if we reenable the feature if it was previously rolled back?**
  There will be no adverse effects of enabling the feature gate after it was
  disabled.

* **Are there any tests for feature enablement/disablement?** No

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?
A rollout can fail on enabling the feature if there is a bug in the node log query code
which can cause the kubelet to crash. However this has not been observed in practice or
in the end to end tests. When the kubelet comes up successfully on enabling the feature,
it will have no impact on workloads.
There should be no impact on rolling back this feature.

###### What specific metrics should inform a rollback?
A kubelet crash on enabling just this feature would be an indicator that a rollback is
required. So far no CPU or memory spikes have been observed on enabling this feature but
that could be another indicator.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?
Yes. The following manual tests were done:
- Brought up a 1.30-alpha cluster without the kubelet feature gate and kubelet option. Enabled it
  the feature and ensured that the feature worked. Disabled the feature and ensured that the
  log proxy endpoint worked as before.
- Brought up a 1.29 cluster and enabled the feature. Upgraded the kubelet to 1.30-alpha and ensured
  that the feature continued to work. Downgraded the kubelet to 1.29 and ensured that the feature
  continued to work. Upgraded the kubelet again to 1.30 and ensured that the feature worked.
- Brought up a 1.29 cluster and enabled the feature. Upgraded the kubelet to 1.30-alpha and ensured
  that the feature continued to work. Disabled the feature and downgraded the kubelet to 1.29 and
  ensured that the log proxy endpoint worked as before. Upgraded the kubelet to 1.30-alpha and
  ensured that the log proxy endpoint worked as before. Enabled the feature again and ensured it worked
  as advertised.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?
No

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?
While this feature does not affect any workloads an operator can determine if this feature
is enabled by checking the kubelet logs for "feature gates: {map[NodeLogQuery:true]}".

###### How can someone using this feature know that it is working for their instance?
- [x] Other
  - Details: The cluster administrator can confirm that this feature works by querying the kubelet log proxy
    endpoint. Example: "kubectl get --raw "/api/v1/nodes/node-1.example/proxy/logs/?query=kubelet"

### Dependencies

* **Does this feature depend on any specific services running in the cluster?**
  - kubelet
    - Usage description:
      - Impact of its outage on the feature: If kubelet is not running on the
        node this feature will not work.
      - Impact of its degraded performance or high-error rates on the feature:
        If the kubelet is degraded this feature will also be degraded i.e. the
        node logs will not be returned.

### Scalability

* **Will enabling / using this feature result in any new API calls?**
  No

* **Will enabling / using this feature result in introducing new API types?**
  The feature does not introduce a new API from an API server perspective but
  the existing kubelet proxy/log endpoint will have new features built into it.

* **Will enabling / using this feature result in any new calls to the cloud
provider?**
  No

* **Will enabling / using this feature result in increasing size or count of
the existing API objects?**
  No

* **Will enabling / using this feature result in increasing time taken by any
operations covered by [existing SLIs/SLOs]?**
  No

* **Will enabling / using this feature result in non-negligible increase of
resource usage (CPU, RAM, disk, IO, ...) in any components?**
  In the case of large logs, there is potential for an increase in RAM and CPU
  usage on the node when an attempt is made to stream them. However, so far no
  CPU or memory spikes have been reported from the field.

### Troubleshooting

## Implementation History

- Created on Jan 14, 2021
- Updated on May 5th, 2021
- Updated on Dec 13th, 2022
- Updated on May 2nd, 2023
- Updated on Feb 5th, 2024

## Drawbacks

## Alternatives

Alternatively we could use a client side reader on the nodes to redirect the
logs. The Windows side would require privileged container support. However this
would not help scenarios where containers are not launching successfully on the
nodes.
