Director
========

This package is built to make it easy to write and to test background
goroutines. These are the kinds of goroutines that are meant to have a
reasonable lifespan centered around a central loop.

The core interface for the package is the `Looper`. Two `Looper`
implementations are currently included, a `TimedLooper` whichs runs the loop on
a specified interval, and a `FreeLooper` which runs the loop as quickly as
possible.

The `Looper` interface looks like this:

```go
type Looper interface {
	Loop(fn func() error)
	Wait() error
	Done(err error)
	Quit()
}
```

Here's an example goroutine that could benefit from a `FreeLooper`:

```go
func RunForever() error {
	for {
		... do some work ...
		if err != nil {
			return err
		}
	}
}

go RunForever()
```

This is totally valid code, but it kind of stinks, because we can't easily test
the code with `go test`. To do that, we need to have a way to get the code to
quit after iterating. So we can do something like this:

```go
func RunForever(quit chan bool) error {
	for {
		... do some work ...
		if err != nil {
			return err
		}

		select {
		case <-quit:
			return nil
		}
	}
}

quit := make(chan bool, 1)
go RunForever(quit)
```

Now we can tell it to quit when we want to. We probably wanted that to begin
with, so that the main program can tell the goroutine to end. But it also now
means we can test it by using a buffered channel, putting a message into the
channel, then running the test.

But what about when we want to run it more than once in a pass? Or when we want
to have our code wait on its completion somewhere during execution? These are
all common patterns and require boilerplate code.  If you do that once in your
program, fine. But it's often the case that this proliferates all over the
code. Instead we could use a `FreeLooper` like this:

```go
func RunForever(looper Looper) error {
	looper.Loop(func() error {
		... do some work ...
		if err != nil {
			return err
		}

		select {
		case <-quit:
			return nil
		}
	})
}

looper := NewFreeLooper(1, make(chan error))
go RunForever(looper)

err := looper.Wait()
if err != nil {
	... handle it ...
}
```

That will run the loop once, and wait for it to complete, handling the
resulting error.

Or we, can tell it to run forever, and then stop it when we want to:

```go
looper := NewFreeLooper(FOREVER, make(chan error))
go RunForever(looper)

... do more work ...

looper.Quit()
err := looper.Wait()
if err != nil {
	... handle it ...
}

```

And if on a later basis we want this to run as a timed loop, such as one
iteration per 5 seconds, we can just substitute a `TimedLooper`:

```go
looper := NewTimedLooper(1, 5 * time.Second, make(chan error))
go RunForever(looper)
```
