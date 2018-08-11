Mocking Go's Time Pacakge with Crock and TimeProxy
==================================================

by Eric Chlebek, Aug. 2018

Introduction
------------

Unit testing is one of the most widely used techniques in software development
today. Software developers decompose their programs into isolated pieces, and
test that these pieces fulfill their business purpose. Unit testing has enjoyed
a wide array of advocates, and there has been much written on the topic.

In practice, many software developers still struggle with unit testing, despite
good intentions. Their unit tests sometimes turn out to be integration tests.
This is caused by a high degree of coupling between components. A unit of code
is coupled with another unit of code if it is dependent upon it to function. No
other unit of code could have been used instead.

While coupling is almost always preventable through lucid, well thought-out
design, it's common for software developers to write highly coupled code by
accident. This happens when software developers do not fully consider a unit
of code's dependencies. Programming, a complex task, can make this mistake
happen easyily, especially when working with components one is not familiar
with, or which are intended to be used in a way that increases coupling.

In Go, interfaces provide a method of decoupling. The `io.Reader` interface,
for instance, can represent a network socket, a file, or a string. Interfaces
are an excellent way to inject dependencies into code units that requires access
to complex, stateful objects. These objects can be replaced in unit tests
by simple, static objects, while maintaining the interface contract.


Go's Time Library
-----------------

Unfortunately, Go's `time` library does not fit this pattern very well. The
library has several package-scoped functions which are stateful, which makes
creating a mock implementation difficult. If users call package-scoped
functions, they are always getting the real deal. There is no way to replace,
or patch, these functions at run-time.

Programs that use the `time` library's functions become difficult to test. How
can a piece of code that schedules events to occur in the future be effectively
tested to ensure that the event will fire when that future moment arrives?
Beyond waiting for the moment to arrive, testing in this scenario always feels
a bit synthetic. Some programmers will test this scenario by using a very short
window of time and waiting, but this can often lead to tests that yield false
positives.

One good solution is to use a proxy object that can dispatch calls to the real
time library in production, and be overridden in testing. But this can require
substantial refactoring. In existing codebases, sometimes a lower effort
approach is better, and presents a lower degree of difficulty for the
maintainer.


Replacing Go's Time Library With TimeProxy
------------------------------------------

```go
// Package timeproxy contains functions and data types for working with time.
// The package exists so that time functionality can be mocked in tests.
package timeproxy
```

[TimeProxy](https://godoc.org/github.com/echlebek/timeproxy) is a library that
can provide a proxy object for time, and can also be used as a drop-in
replacement for stdlib `time`, by aliasing `time`'s types and offering the same
API as `time`. [TimeProxy](https://godoc.org/github.com/echlebek/timeproxy) can
be used anywhere stdlib `time` is used, like so:

```diff
 import (
-       "time"
+       time "github.com/echlebek/timeproxy"
 )
```

The `timeproxy` package has all the same types and functions as the standard
library `time` package. It uses Go type aliases so that types declared in the
`timeproxy` package are the same types as in the stdlib `time` package.

By default, a program that uses `timeproxy` instead of `time` is guaranteed to
behave in the same way. `timeproxy` simply dispatches all calls to the `time`
library.

The difference is that `timeproxy`'s functions call methods on a struct within
the `timeproxy` package called `RealTime`. For example, here is an excerpt
from the `timeproxy` package implementation:

```go
// RealTime dispatches all calls to the stdlib time package
type RealTime struct{}

// TimeProxy is an interface that maps 1:1 with the standalone functions in
// this pacakge. By default, TimeProxy uses the functions from the time package
// in the standard library.
//
// Replace TimeProxy with a different implementation to gain control over time
// in tests.
var TimeProxy Proxy = RealTime{}

// Since calls time.Since
func (RealTime) Since(t Time) Duration {
	return time.Since(t)
}

// Since calls TimeProxy.Since
func Since(t Time) Duration {
	return TimeProxy.Since(t)
}
```

By using indirection, the `timeproxy` library creates an opportunity to
override the behaviour of its package-level functions by replacing the
implementation of `TimeProxy` with another instance of `Proxy`.

Let's have a look at the `Proxy` interface.

```go
// Proxy is a proxy for the time package. By default, the methods from
// stdlib are called.
type Proxy interface {
	Now() Time
	Tick(Duration) <-chan Time
	After(Duration) <-chan Time
	Sleep(Duration)
	NewTicker(Duration) *Ticker
	AfterFunc(Duration, func()) *Timer
	NewTimer(Duration) *Timer
	Since(Time) Duration
	Until(Time) Duration
}
```

Notice that the `Proxy` interface specifies most of the functions from the
stdlib `time` package. The only ones that are not present are functions like
`Unix` and `Parse`. These functions are pure functions, and because of their
stateless nature, they require no replacement in unit tests.

It should be apparent by now that developers can use the `timeproxy` library
anywhere they would use the stdlib `time` library. But now what? Without another
implementation of the `timeproxy.Proxy` interface, this alone will not help
very much when working with unit tests.


Crock, a Fake Time Implementation
---------------------------------

The [Crock](https://godoc.org/github.com/echlebek/crock) library provides an
implementation of `timeproxy.Proxy`, suitable for use in unit testing. Using
[Crock](https://godoc.org/github.com/echlebek/crock) is easy; just replace the
`TimeProxy` variable in the tests' `init` function.

```go
import (
	"github.com/echlebek/crock"
	time "github.com/echlebek/timeproxy"
)

func init() {
	time.TimeProxy = crock.NewTime(time.Unix(0, 0))
}
```

In packages where stdlib `time` has been replaced with `timeproxy`, this will
result in all `time` calls being dispatched to a `crock.Time` object. Note that
the `crock.NewTime` takes a `time.Time`; this is to establish when the current
time should be set. In this example, it's been set to the start of the [Unix
epoch](https://en.wikipedia.org/wiki/Unix_time).

Once the time implementation has been replaced, crock gives us the flexibility
to do whatever we need to do. Time can be started, stopped, fast-forwarded,
or set to any valid time in the past or future. The `Ticker`s and `Timer`s
generated by the library will behave accordingly.

Here's a toy example - a ticker is set to tick every 1000 hours. With `crock`,
we don't need to wait to observe the tick.

```go
package main

import (
	"testing"

	"github.com/echlebek/crock"
	time "github.com/echlebek/timeproxy"
)

var testTime = crock.NewTime(time.Unix(0, 0))

func init() {
	time.TimeProxy = testTime
}

func TestVeryLongTick(t *testing.T) {
	ticker := time.NewTicker(time.Hour * 1000)
	defer ticker.Stop()

	testTime.Set(time.Now().Add(time.Hour * 1001))
	<-ticker.C
}
```

The test runs almost instantly. Another thing that we can do is speed up the
passage of time. This test also runs almost immediately.

```go
func TestVeryLongTicks(t *testing.T) {
	// Time passes 100,000 times faster than normal
	testTime.Multiplier = 100000

	// Time resolution, in terms of real time. This fundamentally limits the
	// number of actual ticks that can be observed.
	testTime.Resolution = time.Millisecond

	ticker := time.NewTicker(time.Hour)
	defer ticker.Stop()

	// Start the flow of time
	testTime.Start()
	defer testTime.Stop()

	done := time.After(time.Hour * 10)

	for {
		select {
		case <-done:
			return
		case <-ticker.C:
		}
	}
}
```


Drawbacks and Caveats
---------------------

One issue with a global variable, which `timeproxy.TimeProxy` is, is that state
from one test could make its way to another if users aren't careful. This makes
parallel test execution impossible - a clear drawback to the approach. Another
issue is that if time is not stopped between tests, results could be
nondeterministic.

For programs where this is an issue, I'd encourage developers to use the
`timeproxy.Proxy` interface as a parameters to any functions that require the
use of time. `timeproxy.RealTime` can be used in production, and `crock.Time`
can be used in unit tests.

Another issue is that due to the design of the Go `Ticker` and `Timer` types,
no interface could be used to represent these types. These types have a struct
field that stores the channel they send ticks to. Since the channel is accessed
by a struct field instead of a method, it's not possible to represent a `Timer`
or `Ticker` without creating a new type. So `timeproxy` has its own versions
of these types. By default, they simply call a `time.Ticker` or `time.Timer`.


Case Study: Replacing `time` With `timeproxy` in Sensu's `schedulerd` Package
-----------------------------------------------------------------------------

In this [pull request](https://github.com/sensu/sensu-go/pull/1889/files), the
Go stdlib `time` package is replaced with `timeproxy`, and the
`timeproxy.TimeProxy` variable is mocked out with `crock.Time` in unit tests.

The pull request took a bit of fiddling, as this was the first use of crock in
a real application. Some race conditions in `crock` had to be resolved before
all was well. Initially, `crock` was conceived as a library that could be used
in tests only, but it quickly became apparent that a shim was required for
non-test code. Terrance Kennedy aided in the creation of TimeProxy in a quick
pairing session, and the way the two libraries play together became better
defined through trial and error.

All in all, the change was quite successful. The tests became much more
repeatable, and test duration decreased substantially as well.

Over time, it is expected that the rest of the components in Sensu that rely on
the use of stdlib `time` will also be able to benefit from this testing
approach.


Conclusion
----------

The advent of Go type aliases has made gradual code repair possible, and a side
effect of this is that one can gradually repair uses of types from the standard
library. This property is exploited to create an effective tool for diverting
calls from stdlib time to a third-party implementation, for the purpose of unit
testing. In real world use, this has lead to more robust tests which also take
less time to execute.
