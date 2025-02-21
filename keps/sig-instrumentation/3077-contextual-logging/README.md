# [KEP-3077](https://github.com/kubernetes/enhancements/issues/3077): contextual logging

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 3](#story-3)
    - [Story 4](#story-4)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Uninitialized logger](#uninitialized-logger)
    - [Logging during initialization](#logging-during-initialization)
    - [Performance overhead](#performance-overhead)
    - [Transition](#transition)
      - [no logger set or not set with <code>ContextualLogger(true)</code>](#no-logger-set-or-not-set-with-contextualloggertrue)
      - [logger set with <code>ContextualLogger(true)</code>](#logger-set-with-contextualloggertrue)
      - [Removing Kubernetes' dependency on legacy klog features](#removing-kubernetes-dependency-on-legacy-klog-features)
    - [Pitfalls during usage](#pitfalls-during-usage)
      - [Logger in context and function not consistent](#logger-in-context-and-function-not-consistent)
      - [Overwriting context not needed initially, forgotten later](#overwriting-context-not-needed-initially-forgotten-later)
      - [Redundant key/value pairs](#redundant-keyvalue-pairs)
      - [Modifying the same variable in a loop](#modifying-the-same-variable-in-a-loop)
      - [Unused return values](#unused-return-values)
- [Design Details](#design-details)
  - [Feature gate](#feature-gate)
  - [Text format](#text-format)
  - [Default logger](#default-logger)
    - [logging helper API](#logging-helper-api)
    - [Logging in tests](#logging-in-tests)
  - [<a href="https://github.com/kubernetes/klog/tree/main/hack/tools/logcheck">logcheck</a>](#logcheck)
    - [Code examples](#code-examples)
      - [Unit testing](#unit-testing)
      - [Injecting common value, logger passed through existing ctx parameter or new parameter](#injecting-common-value-logger-passed-through-existing-ctx-parameter-or-new-parameter)
      - [Resulting output](#resulting-output)
  - [Integration with log/slog](#integration-with-logslog)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
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
- [Status and next steps](#status-and-next-steps)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Per-component logger](#per-component-logger)
  - [Propagating a logger to init code](#propagating-a-logger-to-init-code)
  - [Panic when FromContext is called before setting a logger](#panic-when-fromcontext-is-called-before-setting-a-logger)
  - [Clean separation of contextual logging and traditional klog logging](#clean-separation-of-contextual-logging-and-traditional-klog-logging)
  - [Use log/slog instead of klog+logr](#use-logslog-instead-of-kloglogr)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [X] (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [X] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [x] (R) Production readiness review completed
- [X] (R) Production readiness review approved
- [X] "Implementation History" section is up-to-date for milestone
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


Contextual logging replaces the global logger by passing a `logr.Logger`
instance into functions via a `context.Context` or an explicit
parameter, building on top of [structured
logging](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/1602-structured-logging).

It enables the caller to:
- attach key/value pairs that get included in all log messages
- add names that describe which component or operation triggered a log messages
- reduce the amount of log messages emitted by a callee by changing the
  verbosity

This works without having to pass any additional information into the callees
because the additional information or settings are stored in the modified
logger instance that is passed to it.

Third-party components that use Kubernetes packages like client-go are no
longer forced to use klog. They can choose an arbitrary implementation of
`logr.Logger` and configure it as desired.

During unit testing, each test case can use its own logger to ensure that log
output is associated with the test case.

## Motivation

### Goals

- Remove direct log calls through the `k8s.io/klog` API and
  the hard dependency on the klog logging implementation from all packages.
- Grant the caller of a function control over logging inside that function,
  either by passing a logger into the function or by configuring the object
  that a method belongs to.
- Provide documentation and helper code for setting up logging in unit tests.
- Change as few exported APIs as possible.

### Non-Goals

- Remove the klog text output format.
- Deprecate klog.

## Proposal

The proposal is to extend the scope of the on-going conversion of logging calls
to structured logging by removing the dependency on the global klog
logger. Like the conversion to structured logging, this activity can convert
code incrementally over as many Kubernetes releases as necessary without
affecting the usage of Kubernetes in the meantime.

Log calls that were already converted to structured logging need to be updated
to use a logger that gets passed into the function in one of two ways:
- as explicit parameter
- attached to a `context.Context`

When a function already accepts a context parameter, then that will be used
instead of adding a separate parameter. This covers most of client-go and
avoids another big breaking change for the community.

When a function does not accept a context parameter but needs a context for
calling some other functions, then a context parameter will be added. As a
positive side effect, such a change can then also remove several `context.TODO`
calls (currently over 6000 under `pkg`, `cmd`, `staging` and `test`). An
explicit logger parameter is suitable for functions which don’t need a context
and never will.

The rationale for not using both context and an explicit logger parameter is
the risk that the caller passes an updated logger which it didn't add to the
context. When the context is then passed down to other functions instead of the
logger, logging in those functions will not produce the expected result. This
could be avoided by carefully reviewing code which modifies loggers, but
designing an API so that such a mistake cannot happen seems safer. The logcheck
linter will check for this. `nolint:logcheck` comments can be used for
those functions where passing both is preferred despite the ambiguity.

`k8s.io/klog` gets extended to support contextual logging: the logger that gets
installed with a new `SetLoggerWithOption(..., ContextualLogger(true))` call can be retrieved and used
directly.  A new `FromContext` function wraps the corresponding function from
`go-logr/logr` and if no logger is set for a context, falls back to logging
through klog with `Logger` as the API that is used by the code which emits
log entries.

To simplify that code, aliases for functions and types from `go-logr/logr` get
added to klog. That way, a single import statement will be enough in most
files.

A feature gate controls whether contextual logging is used or a global logger
is accessed directly.

### User Stories

#### Story 1

kube-scheduler developer Joan [wants to know which
pod](https://github.com/kubernetes/kubernetes/issues/91633#issuecomment-675074671)
and which operation and scheduler plugin log messages are associated with.

When kube-scheduler starts processing a pod, it creates a new logger with
`logger.WithValue("pod", klog.KObj(pod))` and passes that around. While
iterating over plugins in certain operations, another logger gets created with
`logger.WithName(<operation>).WithName(<plugin name>)` and then is used when
invoking that plugin. This adds a prefix to each log message which represents
the call path to the specific log message, like for example
`NominatedPods/Filter/VolumeBinding`.

#### Story 2

Scheduler-plugins developer John wants to increase the verbosity of the
scheduler while it processes a certain pod (["per-flow additional
log"](https://github.com/kubernetes-sigs/scheduler-plugins/pull/289)).

John does that by using `logger.V(0)` as logger for important pods and
`logger.V(2)` as logger for less important ones. Then when the scheduler’s
verbosity threshold is `-v=1`, a log message emitted with `V(1).InfoS` through
the updated logger will be printed for important pods and skipped for less
important ones.

#### Story 3

Kubernetes contributor Patrick is working on a unit test with many different
test cases. To minimize overall test runtime, he allows different test cases to
execute in parallel with `t.Parallel()`. Unfortunately, the code under test
suffers from a rare race condition that is triggered randomly while executing
all tests, but never when running just one test case at a time or when single
stepping through it.

He therefore wants to enable logging so that `go test` shows detailed log
output for a failed test case, and only for that test case. He wants to run it
like that by default in the CI so that when the problem occurs, all information
is immediately available. This is important because the error might not show up
when trying the same test invocation locally.

When everything works, he wants `go test` to hide the log output to avoid
blowing up the size of the CI log files.

For each inner test case he adds a `NewTestContext(t)` invocation and uses the
returned context and logger for that test case.

#### Story 4

Client developer Joan wants to use client-go in her application, but is [less
interested in log messages from
it](https://kubernetes.slack.com/archives/CG3518SFJ/p1634217685020400). Joan
makes the log messages from client-go less verbose by creating a logger with
`logger.V(1)` and passing that to the client-go code. Now a `logger.V(3).Info`
call in client-go is the same as a `logger.V(4).Info` in the application and
will not be shown for `-v=3`.


### Risks and Mitigations

#### Uninitialized logger

One risk is that code uses an uninitialized `logr.Logger`, which would lead to
nil pointer panics. This gets mitigated with the [logr convention](https://github.com/go-logr/logr/blob/ec7c16ccad4699c8ad291c385dbb7a3802c2d01c/logr.go#L118-L125)
that a
`logr.Logger` passed by value always must be usable. When the logger is optional,
this needs to be indicated by passing a pointer.

#### Logging during initialization

Ideally, no log messages should be emitted before the program is done with
setting up logging as intended. This is already [problematic
now](https://github.com/kubernetes/kubernetes/issues/100152) because output may
change from text (the current klog default) to JSON (once initialized). There
will be no automatic mitigation for this. Such log calls will have to be found
and fixed manually, for example by [passing the error back to `main`
instead](https://github.com/kubernetes/kubernetes/pull/104774), which is part
of an effort to [remove the dependency on logging before an unexpected program
exit](https://github.com/kubernetes/kubernetes/issues/102231).

#### Performance overhead

Retrieving a logger from a context on each log call will have a higher overhead
than using a global logger instance. The overhead will be measured for
different scenarios. If a function uses a logger repeatedly, it should retrieve
the logger once and then use that instance.

More expensive than logger lookup are `WithName` and `WithValues` and creating
a new context with the modified logger. This needs to be used cautiously in
performance-sensitive code. A possible compromise is to enhance logging with
such additional information only at higher log levels.

#### Transition

Code that uses traditional klog calls and code that use the new contextual
logging will remain interoperable.

##### no logger set or not set with `ContextualLogger(true)`

`FromContext` returns a klogr instance that writes through klog. The traditional
klog configuration is used (output handling, verbosity).

##### logger set with `ContextualLogger(true)`

The traditional klog API calls were already mapped to `Logger.Info` and
`Logger.Error` for structured logging and the logger handles the output
formatting, nothing changes there.

`FromContext` returns the logger and log entries are emitted directly, without
going through klog.

##### Removing Kubernetes' dependency on legacy klog features

A new logger gets added to klog which supports all non-deprecated klog flags
(i.e. `-v` and `-vmodule`). Once the [deprecation of klog
flags](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/2845-deprecate-klog-specific-flags-in-k8s-components)
is complete, Kubernetes can instantiate that logger in
`k8s.io/component-base/logs` and install it via `SetLogger`. Then all log
output from Kubernetes will be handled without going through any of the code in
`klog.go`.

That file then will still be vendored into `k/k/vendor` because `FromContext`
may have to fall back to it. It just won't get used anymore by those binaries
which use `k8s.io/component-base/logs`. This includes all of the Kubernetes
control plane and kubectl.

If an even cleaner separation is desired, a `k8s.io/klog/v3` could get released
without the legacy code. It would fall back to the stand-alone text logger
instead of klogr. However, that then is a breaking change that currently isn't
planned.

#### Pitfalls during usage

Code reviews must catch the following incorrect patterns.

##### Logger in context and function not consistent

Incorrect:

```go
func foo(ctx context.Context) {
   logger := klog.FromContext(ctx)
   ctx = klog.NewContext(ctx, logger.WithName("foo"))
   doSomething(ctx)
   // BUG: does not include WithName("foo")
   logger.Info("Done")
}
```

Correct:

```go
func foo(ctx context.Context) {
   logger := klog.FromContext(ctx).WithName("foo")
   ctx = klog.NewContext(ctx, logger)
   doSomething(ctx)
   logger.Info("Done")
}
```

In general, manipulating a logger and the corresponding context should be done
in separate lines.

##### Overwriting context not needed initially, forgotten later

Initial, correct code with contextual logging:

```go
func foo(ctx context.Context) {
   logger := klog.FromContext(ctx).WithName("foo")
   doSomething(logger)
   logger.Info("Done")
}
```

A line with `ctx = klog.NewContext(ctx, logger)` could be added above (it
compiles), but it causes unnecessary overhead and some linters complain about
"new value not used".

However, care then must be taken when adding code later which uses `ctx`:

```go
func foo(ctx context.Context) {
   logger := klog.FromContext(ctx).WithName("foo")
   doSomething(logger)
   // BUG: ctx does not contain the modified logger
   doSomethingWithContext(ctx)
   logger.Info("Done")
}
```

##### Redundant key/value pairs

When the caller already adds a certain key/value pair to the logger, the callee
should still add it as parameter in log calls where it is important. That is
because the contextual logging feature might be disabled, in which case adding
the value in the caller is a no-op. Another reason is that it is not always
obvious whether a value is part of the logger or not. Later when the feature
check is removed and it is clear that keys are redundant, they can be removed.

If there are duplicate keys, the text output will only print the value from the
log call itself because that is the newer value. For JSON, zap will format the
duplicates and then log consumers will keep only the newer value because it
comes last, so the end effect is the same.

This situation is similar to wrapping errors: whether an error message contains
information about a parameter (for example, a path name) needs to be documented
to avoid storing redundant information when the caller wraps an error that was
created by the callee.

Analysis of the JSON log files collected at a high log level during a Prow test
run can be used to detect cases where redundant key/value pairs occur in
practice. This is not going to be complete, but it doesn’t have to be because
the additional overhead for redundant key/value pairs matters a lot less when
the message gets emitted infrequently.

##### Modifying the same variable in a loop

Reusing variable names keeps the code readable and prevents errors because the
variable that isn’t meant to be used will be shadowed. However, care must be
taken to really create new variables. This is broken:

```go
func foo(logger klog.Logger, objects ...string) {
    for _, obj := range objects {
        // BUG: logger accumulates different key/value pairs with the same key
        logger = logger.WithValue("obj", obj)
        doSomething(logger, obj)
    }
}
```

A new variable must be created with `:=` inside the loop:

```go
func foo(logger klog.Logger, objects ...string) {
    for _, obj := range objects {
        // This logger variable shadows the function parameter.
        logger := logger.WithValue("obj", obj)
        doSomething(logger, obj)
    }
}
```

##### Unused return values

This code looks like it adds a name, but isn’t doing it correctly:

```go
func foo(logger klog.Logger, objects ...string) {
        // BUG: WithName returns a new logger with the name,
        // but that return value is not used.
        logger.WithName("foo")
        doSomething(logger)
    }
}
```


## Design Details

klog currently provides a formatter for log messages, a global logger instance,
and some helper code. It also contains code for [log
sanitization](https://github.com/kubernetes/enhancements/issues/1753), but that
is a [deprecated](https://github.com/kubernetes/enhancements/pull/3096) alpha
feature and doesn't need to be supported anymore.

### Feature gate

The `k8s.io/klog` package itself cannot check Kubernetes feature gates because
it has to be a stand-alone package with very few dependencies. Therefore it
will have a global boolean for enabling contextual logging that programs with
Kubernetes feature gates must set:

```
// EnableContextualLogging controls whether contextual logging is enabled.
// By default it is enabled. When disabled, FromContext avoids looking up
// the logger in the context and always returns the fallback logger.
// LoggerWithValues, LoggerWithName, and NewContext become no-ops
// and return their input logger respectively context. This may be useful
// to avoid the additional overhead for contextual logging.
//
// Like SetFallbackLogger this must be called during initialization before
// goroutines are started.
func EnableContextualLogging(enabled bool) {
	contextualLoggingEnabled = enabled
}
```

The `ContextualLogging` feature gate will be defined in `k8s.io/component-base`
and will be copied to klogr during the `InitLogs()` invocation that all
Kubernetes commands already go through after their option parsing.

`LoggerWithValues`, `LoggerWithName`, and `NewContext` are helper functions
that wrap the corresponding functionality from `logr`:
```
// LoggerWithValues returns logger.WithValues(...kv) when
// contextual logging is enabled, otherwise the logger.
func LoggerWithValues(logger Logger, kv ...interface{}) Logger {
	if contextualLoggingEnabled {
		return logger.WithValues(kv...)
	}
	return logger
}

// LoggerWithName returns logger.WithName(name) when contextual logging is
// enabled, otherwise the logger.
func LoggerWithName(logger Logger, name string) Logger {
	if contextualLoggingEnabled {
		return logger.WithName(name)
	}
	return logger
}

// NewContext returns logr.NewContext(ctx, logger) when
// contextual logging is enabled, otherwise ctx.
func NewContext(ctx context.Context, logger Logger) context.Context {
	if contextualLoggingEnabled {
		return logr.NewContext(ctx, logger)
	}
	return ctx
}
```

The logcheck static code analysis tool will warn about code in Kubernetes which
calls the underlying functions directly. Once the feature gate is no longer needed,
a global search/replace can remove the usage of these wrapper functions again.

Because the feature gate is off during alpha, log calls have to repeat
important key/value pairs even if those also got passed to `WithValues`:

```
logger := logger.WithValues("pod", klog.KObj(pod))
...
logger.Info("Processing", "pod", klog.KObj(pod))
...
logger.Info("Done", "pod", klog.KObj(pod))
```

Starting with GA, the feature will always be enabled and code can be written
without such duplication:

```
logger := logger.WithValues("pod", klog.KObj(pod))
...
logger.Info("Processing")
...
logger.Info("Done")
```

Documentation of APIs has to make it clear which values will always be included
in log entries and thus don't need to be repeated. If in doubt, repeating them
is okay: the text format will filter out duplicates if log call parameters
overlap with `WithValues` parameters. For performance reasons it will not do
that for duplicates between different `WithValues` calls. In JSON, repeating
keys increases log volume size because there is no de-duplication, but the
semantic is the same ("most recent wins").

### Text format

The formatting and verbosity code will be moved into `internal` packages where
they can be shared between the traditional klog implementation and a new
`go-logr/logr.LogSink` implementation in a `textinglogger` package. That
implementation will produce the same output as klog and support `-v` and
`-vmodule`, the two remaining options from klog that didn't get deprecated.

### Default logger

Adding a logger to the context or as parameter is not required. klog will
manage a global logger that code can look up through klog calls.

The traditional API for setting a logger with `SetLogger` remains unchanged,
with the same semantic. To enable contextual logging, a new call has to be
used. This is necessary to avoid breaking code which uses `SetLogger` to
install a logger that relies on klog for verbosity checks.

```go
var (
	// contextualLoggingEnabled controls whether contextual logging is
	// active. Disabling it may have some small performance benefit.
	contextualLoggingEnabled = true

	// globalLogger is the global Logger chosen by users of klog, nil if
	// none is available.
	globalLogger *Logger

	// contextualLogger defines whether globalLogger may get called
	// directly.
	contextualLogger bool

	// klogLogger is used as fallback for logging through the normal klog code
	// when no Logger is set.
	klogLogger logr.Logger = logr.New(&klogger{})
)

// SetLogger sets a Logger implementation that will be used as backing
// implementation of the traditional klog log calls. klog will do its own
// verbosity checks before calling logger.V().Info. logger.Error is always
// called, regardless of the klog verbosity settings.
//
// If set, all log lines will be suppressed from the regular Output, and
// redirected to the logr implementation.
// Use as:
//   ...
//   klog.SetLogger(zapr.NewLogger(zapLog))
//
// To remove a backing logr implemention, use ClearLogger. Setting an
// empty logger with SetLogger(logr.Logger{}) does not work.
//
// Modifying the logger is not thread-safe and should be done while no other
// goroutines invoke log calls, usually during program initialization.
func SetLogger(logger logr.Logger) {
	globalLogger = &logger
	contextualLogger = false
}

// SetLoggerWithOptions is a more flexible version of SetLogger. Without
// additional options, it behaves exactly like SetLogger. By passing
// ContextualLogger(true) as option, it can be used to set a logger that then
// will also get called directly by applications which retrieve it via
// FromContext, Background, or TODO.
//
// Supporting direct calls is recommended because it avoids the overhead of
// routing log entries through klogr into klog and then into the actual Logger
// backend.
func SetLoggerWithOptions(logger logr.Logger, opts ...LoggerOption) {
...
}

// ContextualLogger determines whether the logger passed to
// SetLoggerWithOptions may also get called directly. Such a logger cannot rely
// on verbosity checking in klog.
func ContextualLogger(enabled bool) LoggerOption {
...
}

```

The API for looking up a logger is new:

```go
// FromContext retrieves a logger set by the caller or, if not set,
// falls back to the program's fallback logger.
func FromContext(ctx context.Context) Logger {
	if contextualLoggingEnabled {
		if logger, err := logr.FromContext(ctx); err == nil {
			return logger
		}
	}

	return Background()
}

// TODO can be used as a last resort by code that has no means of
// receiving a logger from its caller. FromContext or an explicit logger
// parameter should be used instead.
//
// This function may get deprecated at some point when enough code has been
// converted to accepting a logger from the caller and direct access to the
// fallback logger is not needed anymore.
func TODO() Logger {
	return Background()
}

// Background retrieves the fallback logger. It should not be called before
// that logger was initialized by the program and not by code that should
// better receive a logger via its parameters. TODO can be used as a temporary
// solution for such code.
func Background() Logger {
	if globalLogger != nil && contextualLogger {
		return *globalLogger
	}

	return klogLogger
}
```

#### logging helper API

To ensure that a single import is enough in source files, that package will also contain:

```go
type Logger = logr.Logger
var New = logr.New
// plus more as needed...
```

#### Logging in tests

Ginkgo tests do not need to be changed. Because each process only runs one test
at a time, the global default can be set and local Logger instances can be
passed around just as in a normal binary.

Unit tests with `go test` require a bit more work. Each test case must
initialize a `klogr/testing` Logger for its own instance of `testing.T`. The
default log level will be 5, the level
[recommended](https://github.com/kubernetes/community/blob/9406b4352fe2d5810cb21cc3cb059ce5886de157/contributors/devel/sig-instrumentation/logging.md#logging-conventions)
for "the steps leading up to errors and warnings" and "for troubleshooting". It
can be higher than in production binaries because `go test` without `-v` will
not print the output for test cases that succeeded. It will only be shown for
failed test cases, and for those the additional log messages may be useful to
understand the failure. Optionally, logging options can be added to the test
binary to modify this default log level by importing the
`k8s.io/klog/v2/ktesting/init` package:

Example with additional command line options:

```go
import (
    "testing"

    "k8s.io/klog/v2/ktesting"
    _ "k8s.io/klog/v2/ktesting/init"
)

func TestSomething(t *testing.T) {
    // Either return value can be ignored with _ if not needed.
    logger, ctx := ktesting.NewTestContext(t)
            logger.Info("test starts")
            doSomething(ctx)
}
```

Custom log helper code must use the `WithCallStackHelper` method to ensure that
the helper gets skipped during stack unwinding:

```go
func logSomething(logger log.Logger, obj interface{}) {
    helper, logger := logger.WithCallStackHelper()
    helper()
    logger.Info("I am just helping", "obj", obj)
}
```

Skipping multiple stack levels at once via `WithCallDepth` is not working with
loggers that output via `testing.T.Log`. `WithCallDepth` therefore should only
be used by code in `test/e2e` where it can be assumed that the logger is not
using `testing.T`.

### [logcheck](https://github.com/kubernetes/klog/tree/main/hack/tools/logcheck)

That tool is a linter for log calls. It will be updated to:
- detect usage of klog log calls in code that should have been converted to
  contextual logging
- check not only klog calls, but also calls through the `logr.Logger` interface,
- detect direct calls to `WithValue`, `WithName`, and `NewContext` where
  the `klog` wrapper functions should be used instead,
- detect function signatures with both `context.Context` and `logr.Logger`.

#### Code examples

See https://github.com/pohly/kubernetes/compare/master-2022-01-12...pohly:log-contextual-2022-01-12
for the helper code and a tentative conversion of some parts of client-go and
kube-scheduler.

##### Unit testing

Each test case gets its own logger instance that adds log messages to the
output of that test case. The log level can be configured with `-v` and
`-vmodule`.

```diff
diff --git a/pkg/scheduler/framework/plugins/volumebinding/binder_test.go b/pkg/scheduler/framework/plugins/volumebinding/binder_test.go
index 8d45e646112..df6feb561d8 100644
--- a/pkg/scheduler/framework/plugins/volumebinding/binder_test.go
+++ b/pkg/scheduler/framework/plugins/volumebinding/binder_test.go
@@ -44,6 +44,8 @@ import (
        k8stesting "k8s.io/client-go/testing"
        featuregatetesting "k8s.io/component-base/featuregate/testing"
        "k8s.io/klog/v2"
+       "k8s.io/klog/v2/ktesting"
+       _ "k8s.io/klog/v2/ktesting/init"
        "k8s.io/kubernetes/pkg/controller"
        pvtesting "k8s.io/kubernetes/pkg/controller/volume/persistentvolume/testing"
        pvutil "k8s.io/kubernetes/pkg/controller/volume/persistentvolume/util"
@@ -124,10 +126,6 @@ var (
        zone1Labels = map[string]string{v1.LabelFailureDomainBetaZone: "us-east-1", v1.LabelFailureDomainBetaRegion: "us-east-1a"}
 )
 
-func init() {
-       klog.InitFlags(nil)
-}
-
 type testEnv struct {
        client                  clientset.Interface
        reactor                 *pvtesting.VolumeReactor
@@ -144,7 +142,8 @@ type testEnv struct {
        internalCSIStorageCapacityInformer storageinformersv1beta1.CSIStorageCapacityInformer
 }
 
-func newTestBinder(t *testing.T, stopCh <-chan struct{}, csiStorageCapacity ...bool) *testEnv {
+func newTestBinder(t *testing.T, ctx context.Context, csiStorageCapacity ...bool) *testEnv {
+       logger := klog.FromContext(ctx)
        client := &fake.Clientset{}
        reactor := pvtesting.NewVolumeReactor(client, nil, nil, nil)
...
@@ -971,11 +970,12 @@ func TestFindPodVolumesWithoutProvisioning(t *testing.T) {
        }
 
        run := func(t *testing.T, scenario scenarioType, csiStorageCapacity bool, csiDriver *storagev1.CSIDriver) {
-               ctx, cancel := context.WithCancel(context.Background())
+               logger, ctx := ktesting.NewTestContext(t)
+               ctx, cancel := context.WithCancel(ctx)
                defer cancel()
 
                // Setup
-               testEnv := newTestBinder(t, ctx.Done(), csiStorageCapacity)
+               testEnv := newTestBinder(t, ctx, csiStorageCapacity)
                testEnv.initVolumes(scenario.pvs, scenario.pvs)
                if csiDriver != nil {
                        testEnv.addCSIDriver(csiDriver)
```

##### Injecting common value, logger passed through existing ctx parameter or new parameter

```diff
diff --git a/pkg/scheduler/generic_scheduler.go b/pkg/scheduler/generic_scheduler.go
index 8af2f3d160a..57be0f5b705 100644
@@ -80,19 +79,25 @@ type genericScheduler struct {
 
 // snapshot snapshots scheduler cache and node infos for all fit and priority
 // functions.
-func (g *genericScheduler) snapshot() error {
+func (g *genericScheduler) snapshot(logger klog.Logger) error {
        // Used for all fit and priority funcs.
-       return g.cache.UpdateSnapshot(g.nodeInfoSnapshot)
+       return g.cache.UpdateSnapshot(logger, g.nodeInfoSnapshot)
 }
 
 // Schedule tries to schedule the given pod to one of the nodes in the node list.
 // If it succeeds, it will return the name of the node.
 // If it fails, it will return a FitError error with reasons.
+//
+// Schedule itself ensures that the pod name is part of all log entries.
 func (g *genericScheduler) Schedule(ctx context.Context, extenders []framework.Extender, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
        trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
        defer trace.LogIfLong(100 * time.Millisecond)
 
-       if err := g.snapshot(); err != nil {
+       logger := klog.FromContext(ctx)
+       logger = klog.LoggerWithValues(logger, "pod", klog.KObj(pod))
+       ctx = klog.NewContext(ctx, logger)
+
+       if err := g.snapshot(logger); err != nil {
                return result, err
        }
        trace.Step("Snapshotting scheduler cache and node infos done")
```


```diff
@@ -671,7 +692,10 @@ func (f *frameworkImpl) RunFilterPlugins(
        nodeInfo *framework.NodeInfo,
 ) framework.PluginToStatus {
        statuses := make(framework.PluginToStatus)
+       logger := klogr.FromContext(ctx).WithName("Filter").WithValues("pod", klogr.KObj(pod), "node", klogr.KObj(nodeInfo.Node()))
        for _, pl := range f.filterPlugins {
+               logger := logger.WithName(pl.Name())
+               ctx := klogr.NewContext(ctx, logger)
                pluginStatus := f.runFilterPlugin(ctx, pl, state, pod, nodeInfo)
                if !pluginStatus.IsSuccess() {
                        if !pluginStatus.IsUnschedulable() {
```

##### Resulting output

Here is log output from `kube-scheduler -v5` for a Pod with an inline volume which cannot be created because storage is exhausted:

```
I1026 16:21:00.461394  801139 scheduler.go:436] "Attempting to schedule pod" pod="default/my-csi-app-inline-volume"
I1026 16:21:00.461476  801139 binder.go:730] PreFilter/VolumeBinding: "PVC is not bound" pod="default/my-csi-app-inline-volume" pvc="default/my-csi-app-inline-volume-my-csi-volume"
```

Whether the additional `PreFilter/VolumeBinding` prefix is useful enough to
justify the overhead will be determined during code reviews.

The next line is from a file which has not been converted. It’s not clear in which context that message gets emitted:
````
I1026 16:21:00.461619  801139 csi.go:222] "Persistent volume had no name for claim" PVC="default/my-csi-app-inline-volume-my-csi-volume"
````

```
I1026 16:21:00.461647  801139 binder.go:266] NominatedPods/Filter/VolumeBinding: "FindPodVolumes starts" pod="default/my-csi-app-inline-volume" node="127.0.0.1"
I1026 16:21:00.461673  801139 binder.go:842] NominatedPods/Filter/VolumeBinding: "No matching volumes for PVC on node" pod="default/my-csi-app-inline-volume" node="127.0.0.1" default/my-csi-app-inline-volume-my-csi-volume="127.0.0.1"
I1026 16:21:00.461724  801139 binder.go:971] NominatedPods/Filter/VolumeBinding: "Node has no accessible CSIStorageCapacity with enough capacity" pod="default/my-csi-app-inline-volume" node="127.0.0.1" pvc="default/my-csi-app-inline-volume-my-csi-volume" pvcSize=549755813888000 sc="csi-hostpath-fast"
I1026 16:21:00.461817  801139 preemption.go:195] "Preemption will not help schedule pod on any node" pod="default/my-csi-app-inline-volume"
I1026 16:21:00.461886  801139 scheduler.go:464] "Status after running PostFilter plugins for pod" pod="default/my-csi-app-inline-volume" status=&{code:2 reasons:[0/1 nodes are available: 1 Preemption is not helpful for scheduling.] err:<nil> failedPlugin:}
I1026 16:21:00.461918  801139 factory.go:209] "Unable to schedule pod; no fit; waiting" pod="default/my-csi-app-inline-volume" err="0/1 nodes are available: 1 node(s) did not have enough free storage."
```

### Integration with log/slog

[`log/slog`](https://pkg.go.dev/log/slog) got added in Go
1.21. Interoperability with slog is [provided by
logr](https://github.com/go-logr/logr/pull/222). Applications which use slog
can route log output from Kubernetes packages into their `slog.Handler` and
vice versa, as demonstrated with [`component-base/logs`
examples](https://github.com/kubernetes/kubernetes/pull/120696).

### Test Plan

The new code will be covered by unit tests that execute as part of
`pull-kubernetes-unit` and by klog GitHub actions.

Converted components will be tested by exercising them with JSON output at high
log levels to emit as many log messages as possible. Analysis of those logs
will detect duplicate keys that might occur when a caller uses `WithValues` and
a callee adds the same value in a log message. Static code analysis cannot
detect this, or at least not easily.

What it can check is that the individual log calls pass valid key/value pairs
(strings as keys, always a matching value for each key, no duplicate keys).

It can also detect usage of klog in code that should only use contextual
logging.

### Graduation Criteria

#### Alpha

- Common utility code available (logcheck with the additional checks,
  experimental new APIs in `k8s.io/klog/v2`)
- Documentation for developers available at https://github.com/kubernetes/community
- At least kube-scheduler framework and some scheduler plugins (in particular
  volumebinding and nodevolumelimits, the two plugins with the most log calls)
  converted
- Initial e2e tests completed and enabled in Prow

#### Beta

- [All of kube-controller-manager](https://github.com/kubernetes/kubernetes/pull/119250) and some [parts of kube-scheduler](https://github.com/kubernetes/kubernetes/pull/115588) converted (in-tree), conversion of out-of-tree components possible, whether they use pflag ([external-provisioner](https://github.com/kubernetes-csi/external-provisioner/pull/639)] or plain Go flags ([node-driver-registrar](https://github.com/kubernetes-csi/node-driver-registrar/pull/259))
- Gathered feedback from developers and surveys
- New APIs in `k8s.io/klog/v2` no longer marked as experimental

#### GA

- All code in kubernetes/kubernetes converted to contextual logging,
  no dependency on traditional klog calls anymore
- User feedback is addressed
- Allowing time for feedback


**For non-optional features moving to GA, the graduation criteria must include
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md


### Upgrade / Downgrade Strategy

Not applicable. The log output will be determined by what is implemented in the
code that currently runs.

### Version Skew Strategy

Not applicable. The log output format is the same as before, therefore other
components are not affected.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

This is not a feature in the traditional sense. Code changes like adding
additional parameters to functions are always present once they are made.  But
some of the overhead at runtime can be eliminated via the `ContextualLogging`
feature gate.

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate
  - Feature gate name: ContextualLogging
  - Components depending on the feature gate: all core Kubernetes components
    (kube-apiserver, kube-controller-manager, etc.) but also several other
    in-tree commands and the test/e2e suite.

###### Does enabling the feature change any default behavior?

No. Unless log messages get intentionally enhanced as part of touching the
code, the log output will be exactly the same as before.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes, by changing the feature gate.

###### What happens if we reenable the feature if it was previously rolled back?

Previous state is irrelevant, so a previous rollback has no effect.

###### Are there any tests for feature enablement/disablement?

Unit tests will be added.

### Rollout, Upgrade and Rollback Planning

Nothing special needed. The same logging flags as before will be supported.

###### How can a rollout or rollback fail? Can it impact already running workloads?

The worst case would be that a null logger instance somehow gets passed into a
function and then causes log calls to crash. The design of the APIs makes that
very unlikely and code reviews should be able to catch the code that causes
this.

###### What specific metrics should inform a rollback?

Components start to crash with a nil pointer panic in the `logr` package.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Not applicable.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

The feature is in use if using the code from a version with support for
contextual logging, since this can't currently be disabled.

###### How can someone using this feature know that it is working for their instance?

Logs should be identical as previously, unless enriched with additional
context, in which case, additional information is available in logs.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Performance should be similar to logging through klog, with overhead for
passing around a logger not exceeding 2%.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [X] Other
  - Details: CPU resource utilization of components with support for contextual logging before/after an upgrade

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

Counting log calls and their kind (through klog vs. logger, `Info`
vs. `Error`) would be possible, but then cause overhead by itself with
questionable usefulness.

### Dependencies

None.

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

Same as before.

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

Pod scheduling (= "startup latency of schedulable stateless pods" SLI) might
become slightly worse.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Initial micro benchmarking shows that function call overhead increases. This is
not expected to be measurable during realistic workloads.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

Works normally.

###### What are other known failure modes?

None besides bugs that could cause a program to panic (null logger).

###### What steps should be taken if SLOs are not being met to determine the problem?

A cluster operator can disable the feature via the feature gate.

Kubernetes developers can revert individual commits that changed log calls once
it has been determined that they introduce too much overhead.

## Implementation History

* Kubernetes 1.24: initial alpha
* Kubernetes 1.27: parts of kube-controller-manager converted
* Kubernetes 1.28: kube-controller-manager converted completely, relationship
  with log/slog in Go 1.21 clarified
* Kubernetes 1.29: kube-scheduler converted completely

## Status and next steps

As of Kubernetes 1.29.1, kube-controller-manager and kube-scheduler have been
converted. The logcheck tool can be used to count remaining log calls that need
to be converted:

```
go install sigs.k8s.io/logtools/logcheck@latest

echo "Component | Non-Structured Logging | Non-Contextual Logging " && echo "------ | ------- |  -------" && for i in $(find pkg/* cmd/* staging/src/k8s.io/* -maxdepth 0 -type d | sort); do  echo "$i | $(cd $i; ${GOPATH}/bin/logcheck -check-structured -check-deprecations=false 2>&1 ./... | wc -l ) | $(cd $i; ${GOPATH}/bin/logcheck -check-structured -check-deprecations=false -check-contextual ./... 2>&1 | wc -l )"; done
```

Note that this also counts calls where it was decided to not convert them. The
actual check with golangci-lint ignores those because of a `//nolint:logcheck`
suppression comment.

Component | Non-Structured Logging | Non-Contextual Logging 
------ | ------- |  -------
cmd/clicheck | 0 | 0
cmd/cloud-controller-manager | 6 | 8
cmd/dependencycheck | 0 | 0
cmd/dependencyverifier | 0 | 0
cmd/fieldnamedocscheck | 1 | 1
cmd/gendocs | 0 | 0
cmd/genkubedocs | 0 | 0
cmd/genman | 0 | 0
cmd/genswaggertypedocs | 2 | 2
cmd/genutils | 0 | 0
cmd/genyaml | 0 | 0
cmd/gotemplate | 0 | 0
cmd/importverifier | 0 | 0
cmd/kubeadm | 264 | 463
cmd/kube-apiserver | 6 | 7
cmd/kube-controller-manager | 0 | 0
cmd/kubectl | 0 | 0
cmd/kubectl-convert | 0 | 0
cmd/kubelet | 0 | 52
cmd/kubemark | 1 | 1
cmd/kube-proxy | 0 | 42
cmd/kube-scheduler | 0 | 0
cmd/preferredimports | 0 | 0
cmd/prune-junit-xml | 0 | 0
cmd/yamlfmt | 0 | 0
pkg/api | 0 | 0
pkg/apis | 0 | 0
pkg/auth | 1 | 1
pkg/capabilities | 0 | 0
pkg/client | 0 | 0
pkg/cloudprovider | 0 | 0
pkg/cluster | 0 | 0
pkg/controller | 0 | 3
pkg/controlplane | 53 | 69
pkg/credentialprovider | 48 | 77
pkg/features | 0 | 0
pkg/fieldpath | 0 | 0
pkg/generated | 0 | 0
pkg/kubeapiserver | 4 | 4
pkg/kubectl | 1 | 2
pkg/kubelet | 2 | 1983
pkg/kubemark | 7 | 7
pkg/printers | 0 | 0
pkg/probe | 7 | 24
pkg/proxy | 0 | 360
pkg/quota | 0 | 0
pkg/registry | 46 | 99
pkg/routes | 2 | 2
pkg/scheduler | 0 | 0
pkg/security | 0 | 0
pkg/securitycontext | 0 | 0
pkg/serviceaccount | 25 | 44
pkg/util | 20 | 57
pkg/volume | 704 | 1110
pkg/windows | 1 | 1
staging/src/k8s.io/api | 0 | 0
staging/src/k8s.io/apiextensions-apiserver | 58 | 89
staging/src/k8s.io/apimachinery | 80 | 125
staging/src/k8s.io/apiserver | 285 | 655
staging/src/k8s.io/client-go | 163 | 283
staging/src/k8s.io/cli-runtime | 1 | 2
staging/src/k8s.io/cloud-provider | 122 | 162
staging/src/k8s.io/cluster-bootstrap | 2 | 4
staging/src/k8s.io/code-generator | 108 | 155
staging/src/k8s.io/component-base | 33 | 64
staging/src/k8s.io/component-helpers | 2 | 4
staging/src/k8s.io/controller-manager | 10 | 10
staging/src/k8s.io/cri-api | 0 | 0
staging/src/k8s.io/csi-translation-lib | 3 | 4
staging/src/k8s.io/dynamic-resource-allocation | 0 | 0
staging/src/k8s.io/endpointslice | 0 | 0
staging/src/k8s.io/kms | 0 | 0
staging/src/k8s.io/kube-aggregator | 45 | 62
staging/src/k8s.io/kube-controller-manager | 0 | 0
staging/src/k8s.io/kubectl | 96 | 160
staging/src/k8s.io/kubelet | 0 | 32
staging/src/k8s.io/kube-proxy | 0 | 0
staging/src/k8s.io/kube-scheduler | 0 | 0
staging/src/k8s.io/legacy-cloud-providers | 1281 | 2015
staging/src/k8s.io/metrics | 0 | 0
staging/src/k8s.io/mount-utils | 55 | 95
staging/src/k8s.io/pod-security-admission | 0 | 1
staging/src/k8s.io/sample-apiserver | 0 | 0
staging/src/k8s.io/sample-cli-plugin | 0 | 0
staging/src/k8s.io/sample-controller | 0 | 0

For Kubernetes 1.30, the focus is on client-go. APIs need to be extended
carefully without breaking existing code so that a context can be provided for
log calls. In some cases, this also makes a context available to code which
currently uses `context.TODO` as a stop-gap measure. Currently there are over
300 of those in `staging/src/k8s.io/client-go`. Whenever new APIs get
introduced, components which were already converted to contextual logging get
updated to use those.

## Drawbacks

Supporting contextual logging is a key design decision that has implications
for all packages in Kubernetes. They don’t have to be converted all at once,
but eventually they should be for the sake of completeness. This may depend on
API changes.

The overhead for the project in terms of PRs that need to be reviewed can be
minimized by combining the conversion to contextual logging with the conversion
to structured logging because both need to rewrite the same log calls.

## Alternatives

### Per-component logger

A logger could be set for object instances and then all methods of that object
could use that logger. This approach is sufficient to get rid of the global
logger and thus for the testing use case. It has the advantage that log
messages can be associated with the object that emits them.

The disadvantage is that associating the log message with the call chain via
multiple `WithName` calls becomes impossible (mutually exclusive designs).

Enriching log messages with additional values from the call chain’s context is
an unsolved problem. A proposal for passing a context to the logger and then
letting the logger extract additional values was discussed in
https://github.com/go-logr/logr/issues/116. Such an approach is problematic
because it increases coupling between unrelated components, doesn’t work for
code which uses the current logr API, and cannot handle values that weren’t
attached to a context.

Finally, a decision on how to pass a logger instance into stand-alone functions
is still needed.

### Propagating a logger to init code

controller-runtime handles the case of init functions retrieving a logger and
keeping that copy for later logging by handing out [a
proxy](https://github.com/kubernetes-sigs/controller-runtime/blob/78ce10e2ebad9205eff8429c3f0556788d680c27/pkg/log/deleg.go).

This has additional overhead (mutex locking, additional function calls for each
log message). Initialization of log output for individual test cases in a unit
test cannot be done this way.

It's better to avoid doing anything with logging entirely in init code.

### Panic when FromContext is called before setting a logger

This would provide an even more obvious hint that the program isn’t working as
intended. However, the log call which triggers that might not always be
executed during program startup, which would cause problems when it occurs in
production. Therefore klog falls back to the traditional klog logging instead.

### Clean separation of contextual logging and traditional klog logging

The initial revision of this KEP described a plan for moving all code for
contextual logging into a `k8s.io/klogr` repository. Transitioning to that
would have removed all legacy code from Kubernetes. However, that transition
would have been complicated and forced all consumers of Kubernetes code to
adjust their code. Therefore the scope of the KEP was reduced from "remove
dependency on klog" to "remove dependency on global logger in klog".

### Use log/slog instead of klog+logr

This isn't viable because `slog` doesn't provide a mechanism to pass a logger
through a context. Therefore it would not be possible to support contextual
logging in packages like client-go where adding an explicit logger parameter
would be a major API break.
