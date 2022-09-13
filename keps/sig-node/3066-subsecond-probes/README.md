<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.** sig-node
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements** is https://github.com/kubernetes/kubernetes/issues/76951
- [ ] **Make a copy of this template directory.**
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# KEP-3066: Subsecond Probes

Probe timeouts are limited to seconds and that does NOT work well for clients looking for finer and coarser grained timeouts.
<!--
Generate TOC with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
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

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. KEP editors and SIG Docs
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->

The Probe struct contains int32 fields that specify seconds for timeouts.
Some users would like to have timeouts less than one second.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

#### Knative

Knative will create Pods (via Deployment) and wait for them to become `Ready`
when handling an HTTP request from an end-user. In this case, Pod readiness
latency has a direct impact on HTTP request latency. (In the steady state,
Knative will re-use an existing Pod, but this situation can happen on scale-up,
and is guaranteed to happen on scale-from-zero.)

### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->
An ability to specify timeouts that are less than one second.
Add additional tests cases to the timeout test cases.


### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->

V2 API for existing objects.
Converting fields from `int32` to `resource.Quantity`.


## Proposal

Add three new fields (of type `int32`) to the Probe struct, which would be used to handle time values in milliseconds: 

- `periodSecondsMilliseconds`
- `initialDelaySecondsMilliseconds`
- `timeoutSecondsMilliseconds`

The seconds and milliseconds fields would be summed to get the appropriate duration. For example,

```
...
periodSeconds: 1
periodMilliseconds: 500
...
```

would be considered a period value of 1.5s or 1500ms. These values are used as `time.Duration` within the prober package ([ex1](https://github.com/kubernetes/kubernetes/blob/a750d8054a6cb3167f495829ce3e77ab0ccca48e/pkg/kubelet/prober/prober.go#L161) [ex2](https://github.com/kubernetes/kubernetes/blob/a750d8054a6cb3167f495829ce3e77ab0ccca48e/pkg/kubelet/prober/worker.go#L132) ), so there's no real difference between 1.5s and 1500ms.

A few additional examples:

`periodSeconds: 2` and `periodMilliseconds: -500` would be 0.5s / 500ms.

`periodSeconds: 1` and `periodMilliseconds: -900` would be 0.1s / 100ms.

More generally, the effective periodSeconds value would be = `periodSeconds` + `periodMilliseconds`.

There are a few corner cases around default values:

`periodSeconds: 0` and `perMilliseconds: 500` would be 10.5s / 10500ms (as 0 => 10 for `periodSeconds` via [defaults.go](https://github.com/kubernetes/kubernetes/blob/3b13e9445a3bf86c94781c898f224e6690399178/pkg/apis/core/v1/defaults.go#L213-L215))

`timeoutSeconds: 0` and `timeoutMilliseconds: 500` would be 1.5s / 1500ms (as 0 => 1 for `timeoutSeconds` via [defaults.go](https://github.com/kubernetes/kubernetes/blob/3b13e9445a3bf86c94781c898f224e6690399178/pkg/apis/core/v1/defaults.go#L210-L212))

Each millisecond field would be restricted to a value less than `1000` (i.e. less than 1 second), and the adjusted interval would never be allowed to be less than 0 (if it was less than 0, it would get failed at the validation stage).


### User Stories (Optional)

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Knative

See the discussion from the Sept 14, 2021 SIG-Node meeting
[recording](https://youtu.be/LMh7c9e7H-Q?list=PL69nYSiGNLP1wJPj5DYWXjiArF-MJ5fNG&t=1049)
[meeting notes](https://docs.google.com/document/d/1Ne57gvidMEWXR70OxxnRkYquAoMpt56o75oZtg-OeBg/edit#heading=h.by9uk7onna00)

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

Changing defaults is a strict no-go.

Accidentally setting a timeout too low could DOS kubelet if many are used.
Mitigate by preventing timeout values too small.
Could be configurable,
100ms is a first guess, but could be adjusted based on user feedback during alpha and performance testing. Could also implement a scaling repeat to reduce risk of thrashing.

Benchmarking currently WIP for alpha, will be implemented for beta to determine the ideal floor. Several benchmarking tools are being considered (options include ones developed by Knative, Amazon, Red Hat). Benchmarking tooling will also work for similar areas (such as the PLEG work.)

Question raised of how probes should be charged: charging is an issue for all exec/attach/port forward requests, and as usual the process in the container should be charged to the container/pod. This KEP isn't looking to make any major architectural changes in this regard.


#### Overriding an existing field

How does the change to overriding a field effect the users of the existing field.

Currently, the time fields in the Probe struct only use values in seconds. Since this KEP proposes allowing a change of the units on those fields, that could be considered an override. 

However, this change would be done on a probe by probe basis and would be opt-in. Meaning that the default behavior would not change unless a user specifically added the millisecond fields to their spec. Absent that specific opt-in by using the new fields, existing users should not be impacted.

## Design Details

Potentially proposed changes as implemented code:
https://github.com/kubernetes/kubernetes/pull/107958/commits

A quick highlighting of key points:

* Implementing a feature gate (`SubSecondProbes`) to gate the feature

* Adding three additional, optional fields to the Probe struct type:

```
PeriodMilliseconds *int32
InitialDelayMilliseconds *int32
TimeoutMilliseconds *int32
```

* Adding a utility function to get a single duration from second / millisecond values

```
// GetProbeTimeDuration combines second and millisecond time increments into a single time.Duration
func GetProbeTimeDuration(seconds int32, milliseconds *int32) time.Duration {
	if milliseconds != nil {
		return time.Duration(seconds)*time.Second + time.Duration(*milliseconds)*time.Millisecond
	}
	return time.Duration(seconds) * time.Second
```

This would be substituted for use in the various places where times are used in `pkg/kubelet/prober`, for example:

```
-	timeout := time.Duration(p.TimeoutSeconds) * time.Second                      // existing code, to be replaced
+	timeout := GetProbeTimeDuration(p.TimeoutSeconds, p.TimeoutMilliseconds)      // new usage, replaces line above
```


<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Existing Struct
https://github.com/kubernetes/kubernetes/blob/master/pkg/apis/core/types.go

https://github.com/kubernetes/kubernetes/blob/7e7bc6d53b021be6fe3d5a1125a990913b7a9028/pkg/apis/core/types.go#L2062-L2094

```
// Probe describes a health check to be performed against a container to determine whether it is
// alive or ready to receive traffic.
type Probe struct {
	// The action taken to determine the health of a container
	ProbeHandler
	// Length of time before health checking is activated.  In seconds.
	// +optional
	InitialDelaySeconds int32
	// Length of time before health checking times out.  In seconds.
	// +optional
	TimeoutSeconds int32
	// How often (in seconds) to perform the probe.
	// +optional
	PeriodSeconds int32
	// Minimum consecutive successes for the probe to be considered successful after having failed.
	// Must be 1 for liveness and startup.
	// +optional
	SuccessThreshold int32
	// Minimum consecutive failures for the probe to be considered failed after having succeeded.
	// +optional
	FailureThreshold int32
	// Optional duration in seconds the pod needs to terminate gracefully upon probe failure.
	// The grace period is the duration in seconds after the processes running in the pod are sent
	// a termination signal and the time when the processes are forcibly halted with a kill signal.
	// Set this value longer than the expected cleanup time for your process.
	// If this value is nil, the pod's terminationGracePeriodSeconds will be used. Otherwise, this
	// value overrides the value provided by the pod spec.
	// Value must be non-negative integer. The value zero indicates stop immediately via
	// the kill signal (no opportunity to shut down).
	// This is a beta field and requires enabling ProbeTerminationGracePeriod feature gate.
	// +optional
	TerminationGracePeriodSeconds *int64
}
```

### Existing Behavior

Taking the Seconds fields, in order of the struct.
InitialDelaySeconds
TimeoutSeconds
PeriodSeconds

#### Defaulting logic
 - InitialDelaySeconds, has no defaulting logic.
   Therefore it would get the golang defaulting logic, and default to 0.
   There is no particular reason to have an initial delay that is millisecond based.
   Timing is not a good way to do sequencing.
   1 second, 2 seconds, etc.
 - TimeoutSeconds, defaults to 1 seconds if unset (which includes the explicitly set to 0 state). This is the last ending time before the timeout fails. Slow Failures.
https://github.com/kubernetes/kubernetes/blob/e19964183377d0ec2052d1f1fa930c4d7575bd50/pkg/apis/core/v1/defaults.go#L224-L226
```
 	if obj.TimeoutSeconds == 0 {
		obj.TimeoutSeconds = 1
	}
```
 - Period Seconds is the biggest hurdle, but also the most useful.
PeriodSeconds defaults to 10 seconds if unset (which includes the explicitly set to 0 state). Fast Successes.
https://github.com/kubernetes/kubernetes/blob/e19964183377d0ec2052d1f1fa930c4d7575bd50/pkg/apis/core/v1/defaults.go#L227-L229
```
	if obj.PeriodSeconds == 0 {
		obj.PeriodSeconds = 10
	}
```


#### Validation of fields

Must be non-negative, Zero or greater.
```
	allErrs = append(allErrs, ValidateNonnegativeField(int64(probe.InitialDelaySeconds), fldPath.Child("initialDelaySeconds"))...)
	allErrs = append(allErrs, ValidateNonnegativeField(int64(probe.TimeoutSeconds), fldPath.Child("timeoutSeconds"))...)
	allErrs = append(allErrs, ValidateNonnegativeField(int64(probe.PeriodSeconds), fldPath.Child("periodSeconds"))...)

```

#### Fields to Add
What fields may be necessary to add?

In pkg/apis/core/types.go

```

type Probe struct {
    ...
    ...
    // How often (in milliseconds) to perform the probe.
	// +optional
	PeriodMilliseconds *int32
	// Length of time (in milliseconds) before health checking is activated.
	// +optional
	InitialDelayMilliseconds *int32
	// Length of time (in milliseconds) before health checking times out.
	// +optional
	TimeoutMilliseconds *int32
}

```
#### Logic for Added Fields
What is the least logic that could be used?

The combined value of `periodSeconds` and `periodMilliseconds` must be greater than 100ms.
The combined value of `timeoutSeconds` and `timeoutMilliseconds` must be greater than 100ms.
The combined value of `intialDelaySeconds` and `initialDelayMilliseconds` must be greater than 0ms.


### Existing use of Probe struct fields.

#### InitialDelaySeconds
https://github.com/kubernetes/kubernetes/blob/7c2e6125694e1aadc78a5fed1cf696872af50a5e/pkg/kubelet/prober/worker.go#L246-L249
```
	// Probe disabled for InitialDelaySeconds.
	if int32(time.Since(c.State.Running.StartedAt.Time).Seconds()) < w.spec.InitialDelaySeconds {
		return true
	}
```

#### TimeoutSeconds
https://github.com/kubernetes/kubernetes/blob/7c2e6125694e1aadc78a5fed1cf696872af50a5e/pkg/kubelet/prober/prober.go#L160-L213
```
	timeout := time.Duration(p.TimeoutSeconds) * time.Second
```

#### ProbeSeconds
https://github.com/kubernetes/kubernetes/blob/7c2e6125694e1aadc78a5fed1cf696872af50a5e/pkg/kubelet/prober/worker.go#L131-L167
```
func (w *worker) run() {
	probeTickerPeriod := time.Duration(w.spec.PeriodSeconds) * time.Second

	// If kubelet restarted the probes could be started in rapid succession.
	// Let the worker wait for a random portion of tickerPeriod before probing.
	// Do it only if the kubelet has started recently.
	if probeTickerPeriod > time.Since(w.probeManager.start) {
		time.Sleep(time.Duration(rand.Float64() * float64(probeTickerPeriod)))
	}

	probeTicker := time.NewTicker(probeTickerPeriod)

	defer func() {
		// Clean up.
		probeTicker.Stop()
		if !w.containerID.IsEmpty() {
			w.resultsManager.Remove(w.containerID)
		}

		w.probeManager.removeWorker(w.pod.UID, w.container.Name, w.probeType)
		ProberResults.Delete(w.proberResultsSuccessfulMetricLabels)
		ProberResults.Delete(w.proberResultsFailedMetricLabels)
		ProberResults.Delete(w.proberResultsUnknownMetricLabels)
	}()

probeLoop:
	for w.doProbe() {
		// Wait for next probe tick.
		select {
		case <-w.stopCh:
			break probeLoop
		case <-probeTicker.C:
		case <-w.manualTriggerCh:
			// continue
		}
	}
}
```

### Summary

Depending on the importance of the various Probe settings,
it may be best to focus on one field.

The Probe.Period looks to be the most effective to focus on.
Probe.Period describes the 'repeat-rate' for how often a probe will run.

Where Probe.Timeout describes an endpoint for when to stop probing.
Probe.InitialDelay describes how long to wait before starting,
but can be set to zero.


### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation, and anything particularly
challenging to test, should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

Existing unit tests of prober `k8s.io/kubernetes/pkg/kubelet/prober/prober_manager_test.go`.

Existing node-e2e test `k8s.io/kubernetes/test/e2e/common/container_probe.go`

Enhanced with additional test cases, see the buckets added in [the draft implementation](https://github.com/kubernetes/kubernetes/pull/107958/commits)

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. The KEP
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

#### Removing a Deprecated Flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/20190731-production-readiness-review-process.md.

The production readiness review questionnaire must be completed for features in
v1.19 or later, but is non-blocking at this time. That is, approval is not
required in order to be in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.

-->

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**
  - [ X ] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: SubSecondProbes
    - Components depending on the feature gate:
  - [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

* **Does enabling the feature change any default behavior?**
  Any change of default behavior may be surprising to users or break existing
  automations, so be extremely careful here.
  
  No

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?**
  Also set `disable-supported` to `true` or `false` in `kep.yaml`.
  Describe the consequences on existing workloads (e.g., if this is a runtime
  feature, can it break the existing applications?).
  
  Yes, see the [drop test](https://github.com/psschwei/kubernetes/blob/1ca40771e26b96d6121c49b19c2cbe6466694c7a/pkg/api/pod/util_test.go#L1029)

* **What happens if we reenable the feature if it was previously rolled back?**

  No breaking changes

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling or disabling feature
  gates. However, unit tests in each component dealing with managing data, created
  with and without the feature, are necessary. At the very least, think about
  conversion tests if API types are being modified.
  
  Yes, see the [drop test](https://github.com/psschwei/kubernetes/blob/1ca40771e26b96d6121c49b19c2cbe6466694c7a/pkg/api/pod/util_test.go#L1029)

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g., what if some components will restart
   mid-rollout?

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**
  Describe manual testing that was done and the outcomes.
  Longer term, we may want to require automated upgrade/rollback tests, but we
  are missing a bunch of machinery and tooling and can't do that now.

* **Is the rollout accompanied by any deprecations and/or removals of features, APIs,
fields of API types, flags, etc.?**
  Even if applying deprecation policies, they may still surprise some users.

### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
  checking if there are objects with field X set) may be a last resort. Avoid
  logs or events for this purpose.

* **What are the SLIs (Service Level Indicators) an operator can use to determine
the health of the service?**
  - [ ] Metrics
    - Metric name:
    - [Optional] Aggregation method:
    - Components exposing the metric:
  - [ ] Other (treat as last resort)
    - Details:

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  At a high level, this usually will be in the form of "high percentile of SLI
  per day <= X". It's impossible to provide comprehensive guidance, but at the very
  high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code

* **Are there any missing metrics that would be useful to have to improve observability
of this feature?**
  Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
  implementation difficulties, etc.).

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  Think about both cluster-level services (e.g. metrics-server) as well
  as node-level agents (e.g. specific version of CRI). Focus on external or
  optional services that are needed. For example, if this feature depends on
  a cloud provider API, or upon an external software-defined storage or network
  control plane.

  For each of these, fill in the following—thinking about running existing user workloads
  and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?**
  Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
  focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
    
No

* **Will enabling / using this feature result in introducing new API types?**
  Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
  
No

* **Will enabling / using this feature result in any new calls to the cloud
provider?**

No

* **Will enabling / using this feature result in increasing size or count of
the existing API objects?**
  Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)

No

* **Will enabling / using this feature result in increasing time taken by any
operations covered by [existing SLIs/SLOs]?**
  Think about adding additional work or introducing new steps in between
  (e.g. need to do X to start a container), etc. Please describe the details.
  
No

* **Will enabling / using this feature result in non-negligible increase of
resource usage (CPU, RAM, disk, IO, ...) in any components?**
  Things to keep in mind include: additional in-memory state, additional
  non-trivial computations, excessive access to disks (including increased log
  volume), significant amount of data sent and/or received over network, etc.
  This through this both in small and large cases, again with respect to the
  [supported limits].
  
Reducing the probe frequency to subsecond intervals will result in probes polling much more frequently. In a worst case scenario, if all probes are set to the minimum value (100ms), it would result in 10x as many probe runs compared to the current state.

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**
  For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.

* **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->



## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

### `early*Offset`

(copying directly from [Tim's comment](https://github.com/kubernetes/enhancements/pull/3067#issuecomment-1039311016))

Add an extra field to specify “early failure”: an integer quantity of milliseconds (or microseconds - we should leave room for that). That early failure offset is something that legacy clients can ignore without a big problem and would fully support round-tripping to a future Pod v2 API that unifies these fields.

Using a negative offset makes the distinction between (1s - 995ms) more obvious to a legacy client that's unaware of the new fields. A positive offset (0s + 5ms, or maybe 0s + 5000μs) could get interpreted quite differently.

If we do that, we should absolutely disallow a negative offset ≥ 1.0s seconds. Otherwise we have a challenge around unambiguous round-tripping (eg from a future Pod v2 API back to Pod v1).

### `*Duration`

This would be very similar to the previous option, except that it would only require one field instead of multiple ones for each unit type. For example,

```yaml
  periodDuration: 1.5s
```

This could be used in one of two ways:

It could be added to the existing `*Seconds` value, as done in the previous example (i.e. `periodSeconds + periodDuration`). The same caveats listed in the previous option would also apply in this case (signed vs. unsigned, adding vs. subtracting). In this case, the value would be restricted to less than one second.

Alternatively, it could _replace_ the existing `*Seconds` value, using the normalizing pattern described [here](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#making-a-singular-field-plural).

As Tim noted, the current API guidelines [require using integers](https://github.com/kubernetes/community/blob/489de956d7b601fd23c8f48a87b501e0de4a9c7f/contributors/devel/sig-architecture/api-conventions.md?plain=1#L878-L880) for durations. That said, the preceding line also mentions that the best approach is [still TBD](https://github.com/kubernetes/community/blob/489de956d7b601fd23c8f48a87b501e0de4a9c7f/contributors/devel/sig-architecture/api-conventions.md?plain=1#L873-L876), so there's some ambiguity in the docs on this topic.

### A new struct, `ProbeOffset`

This one kind of splits the difference between the two previous options, using a `struct` to allow for specifying a time unit and value:

```golang
type ProbeOffset struct {
  value uint32
  unit string
}
```

which would be used as follows

```yaml
  periodOffset:
    value: 500
    unit: milliseconds
```

This requires introducing a new type for duration, which in hindsight does not meet [either of the API conventions](https://github.com/kubernetes/community/blob/489de956d7b601fd23c8f48a87b501e0de4a9c7f/contributors/devel/sig-architecture/api-conventions.md#units) mentioned above. 

Usage would be similar to the `*Duration` option, i.e. either replace or add to existing field (and if adding to the existing would be restricted to less than 1 second).

### Setting the time units in a different field, `ReadSecondsAs`

The last option would be specify a specific time unit value that would override, in a sense, the "seconds" in all the `*Seconds` fields. For example

```yaml
  periodSeconds: 100
  readSecondsAs: milliseconds
```

would result in an effective period of `500ms`.

Note that this would apply for all `*Seconds` values: if `readSecondsAs` is used, all probe times would need to be rendered in milliseconds. If field level granuality was required (i.e. one value in seconds, one in milliseconds), may as well create `*Milliseconds` fields.

For more examples of how this would work, see [this WIP PR](https://github.com/kubernetes/kubernetes/pull/107958).

Round-tripping issues are handled in a [drop function](https://github.com/kubernetes/kubernetes/blob/39a3c5c880d79fead1ae4dc80f462cac4a33878f/pkg/api/pod/util.go#L607-L636) (using similar logic as would be needed in the first two options).

While this would reduce the number of new fields one has to add, it is to some degree counterintuitive to have a `*Seconds` field in milliseconds.

### v2 api for probe.

I think this means a v2 API for Container therefore Pod.
This seems too invasive.

All Seconds fields become resource.Quantity instead of int32. This supports subdivision in a single field.

Pros:
minimal changes to API (only adding one field)

Cons:
requires conversion back to seconds on legacy versions (does 1.5s round up or down?)

### OffsetMilliseconds

Use a negative offset, and combine with the existing field.

Example:
    Existing Field: int32 PeriodSeconds
    New Field:      int32 PeriodOffsetMilliseconds

    If I want to set to 0.5 seconds, 500 milliseconds,
    PeriodSeconds <= 10 & PeriodOffsetmilliseconds <= -9500
    OR
    PeriodSeconds <= 1 & PeriodOffsetmilliseconds <= -500
    OR
    PeriodSeconds unset, Default of 10, and PeriodOffsetmilliseconds <= -9500

Detail:
Same number of added fields.

Pros:
Uses the existing field, thus readers of only the existing field will still be acting on something that has been logically set. Without the compensation of a negative offset, the output doesn't make much sense as a decision is being made on half of the information, but there is a logical process to why behavior would occur, rather than allowing something to be set before throwing it out.

Cons:
Complicated logic.
Multiple ways to get same resulting time.
Changing behavior in the future involves touching more code than other solutions.

### Reconcile seconds field to nearest whole second.

Minimum of 1 second remains.

Extra logic in defaulter to use the milliseconds field and automatically set the seconds field, allowing those using the seconds field to get something close-enough. This doesn't make much sense for solving rapidity of probes, only for increasing the granularity, such as if I wanted to run

Pros:
Cons:


## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->

Usual infrastructure depending on the complexity of the test cases needed.
