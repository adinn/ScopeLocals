# Scope Locals

## Summary

Enhance the Java API with scope locals, which are dynamically scoped,
effectively final, local variables. They allow a lightweight form of
thread inheritance, which is useful in systems with many threads and
tasks.

ScopeLocal is in essence a redesign of `ThreadLocal` with a more
modern approach. `ThreadLocal` offers unrestricted mutability,
designed back when we thought mutability was a good thing.
`ScopeLocal` restricts mutability, allowing us to reason about the
value more precisely and optimize more effectively.

## History

The need for scope locals arose from Project Loom. Loom enables a way
of Java programming where threads are not a scarce resource to be
carefully managed by thread pools but are much more abundant, limited
only by memory. To allow us to create large numbers of threads &mdash;
potentially millions &mdash; we'll need to make all of the per-thread
structures scale well. Thread-local variables have a significant
time and memory footprint when creating new threads.

When a new `Thread` instance is created, its parent's set of
inheritable thread-local variables is cloned. This is necessary
because a thread's set of thread locals is, by design, mutable, so it
cannot be shared between threads. Every child thread ends up carrying
a local copy of its parent's entire set of thread locals, whether the
child needs them or not.

## Motivation

Thread-local variables are used for a wide variety of reasons in Java
code.

### Transactions of many kinds: the notion of a "current transaction".

    Java Concurrency in Practice:

    "... containers associate a transaction context with an executing
    thread for the duration of an EJB call. This is easily implemented
    using a static Thread-Local holding the transaction context: when
    framework code needs to determine what transaction is currently
    running, it fetches the transaction context from this
    ThreadLocal."

### Hidden parameters for callbacks

You may want to invoke a method `X` in a library that later calls back
into your code. In your callback you need some context, perhaps a
transaction ID or some `File` instances. However, `X` provides no way
to pass a reference through their code into your callback. Set a
thread-local variable, then invoke `X`, then carefully `remove()` the
thread-local variable.

### Thread locals and recursion

Sometimes you want to be able to detect recursion, perhaps because a
framework isn't re-entrant or because you want to limit recursion in
some way, perhaps by counting invocations. A thread-local variable
provides a way to do this: set it once, invoke a method, and test to
see if the thread-local variable is already set. Again, you'll have to
carefully `remove()` the thread-local variable. You'll probably need
the thread-local variable to be a recursion counter, which you can
`remove()` when it gets to zero. The detection of recursion is also
useful in the case of flattened transactions: if a transaction is
already in progress, there's no need to start a new one.

### Caches for Non-thread-safe objects that are expensive to create, e.g. `SimpleDateFormat`.

Some utility classes such as `SimpleDateFormat` are mutable and they
are not thread-safe. The `SimpleDateFormat` API specification suggests
that a new instance should be created for each thread.

We'd like to support as many of these use cases as we can, but only if
the basic properties of effective finality and re-entrancy can be
guaranteed.

## What works with scope locals and what doesn't

### Transactions

These will work fine, but because scope local bindings are always
removed at the end of the scope in which they were bound, you must run
an entire transaction from that binding scope. You won't be able to do
this:

```
try (final Transaction transaction = new Transaction()) {
 // Within this block, every use of Transaction.current()
 // returns the current transaction.
 doSomething();
}
```

but instead you'll have to do this:

```
Transaction.run(() -> { doSomething(); });
```

### Hidden parameters for callbacks

These should work fine.

### Recursion

This case will work fine too, but because scope locals have exactly
the properties required, you won't need a recursion counter.

### Caches for Non-thread-safe objects that are expensive to create

These are problematic for scope locals, perhaps because caches are one
of the few use cases for which thread-local variables are ideally
suited. If you can create what you are likely to need in an outermost
scope, that would work, but it's not ideal.

### The big problem with thread-local variables: thread pools

Thread-local variables don't work in any reasonable way with thread
pools, where threads are shared between tasks. Given that thread pools
are now the dominant way to use threads in Java, we really do need a
way for a task which can be split over worker threads or re-started on
a different one, an effective way of handling some of these use cases
is needed.


## Description

The core idea of scope locals is to support something like a "special
variable" in Common Lisp. This is a dynamically scoped variable, which
acquires a value on entry to a lexical scope; when that scope
terminates, the previous value (or none) is restored. We also support
inheritance for scope locals, so that parallel constructs can set a
value in the parent before sub-tasks start.

One useful way to think of scope locals is as invisible, effectively
final, parameters that are passed through every method invocation.
These parameters will be accessible within the "dynamic scope" of a
scope local's binding operation (the set of methods invoked within the
binding scope, and any methods invoked transitively by them). They are
guaranteed to be re-entrant &mdash; when used correctly.

Scope locals can also provide us with some other nice-to-have
features, in particular:

* **Strong typing** Whenever a scope local is bound to a value, its type
  is checked. This means that a `ClassCastException` is delivered at
  the point an error was made rather than later. Also, there are
  usually more invocations of `get()` than there are binding
  operations, so it makes sense to do the check early.
* **Effective finality** The value bound to a scope local cannot change
  within a method. (It can be re-bound in a callee, of which more
  later.)
* **Well-defined extent** A scope local is bound to a value at the start
  of a scope and its previous value (or none) is always restored at
  the end.
* **Optimization opportunities** These properties allow us to generate
  good code. In many cases a scope-local `get()` is as fast as a local
  variable.

### Some examples

```
  // Declare scope locals x and y
  static final ScopeLocal<MyType> x = ScopeLocal.forType(MyType.class);
  static final ScopeLocal<MyType> y = ScopeLocal.forType(MyType.class);

  {
    ScopeLocal.where(x, expr1)
              .where(y, expr2)
              .run(() -> ... code that uses x.get() and y.get() ...);
  }
```

In this example, `run()` is said to "bind" `x` and `y` to the results
of evaluating `expr1` and `expr2` respectively. While the method
`run()` is executing, any calls to `x.get()` and `y.get()` return the
values that have been bound to them. The methods called from `run()`,
and any methods called by them, comprise the dynamic scope of `run()`.
Because scope locals are effectively final, there is no equivalent of
the `ThreadLocal.set()` method.

(Note 1: the declarations of scope locals `x` and `y` have an explicit
type parameter. This allows us to make a strong guarantee that if any
attempt is made to bind an object of the wrong type to a scope local
we'll immediately throw a ClassCastException.)

(Note 2: `x` and `y` have the usual Java access modifiers. Even though
a scope local is implicitly passed to every method in its dynamic
scope, a method will only be able to use `get()` if that scope local's
name is accessible to the method. So, sensitive security information
can be passed through a stack of non-privileged invocations.)

The following example uses a scope local to make credentials available
to callees.

```
    private static final ScopeLocal<Credentials> CREDENTIALS = ScopeLocal.forType(Credentials.class);

    Credentials creds = ...
    ScopeLocal.where(CREDENTIALS, creds).run(() -> {
        :
        Connection connection = connectDatabase();
        :
    });

    Connection connectDatabase() {
        if (CREDENTIALS.get().s != "MySecret") {
            throw new SecurityException("No dice, kid!");
        }
        return new Connection();
    }
```

We also provide a shortcut for the case where only a single scope
local is set:

```
   {
       ScopeLocal.where(x, expr1, (() -> 
           ... code that uses x.get() ...);
   }

```

This is a natural fit for a `record` when you need to share a group of
values, for example:


```
    record Point(int x, int y) {}
    static final ScopeLocal<Point> position = ScopeLocal.forType(Point.class);

    {
        ScopeLocal.where(position, new Point(33, 66),
            () -> System.out.println(position.get().x()));
    }
```

We recommend this form when multiple values are shared with the same
consumer because it's likely to be more efficent and it's clear to the
reader what is intended.

### Shadowing

It is sometimes useful to be able to re-bind an already-bound scope
local. For example, a privileged method may need to connect to a
database with a less-privileged set of credentials, like so:

```
      Credentials creds = CREDENTIALS.get();
      creds = creds.withLowerTrust();
      ScopeLocal.where(CREDENTIALS, creds).run(() -> {
        :
        Connection connection = connectDatabase();
        :
      });
```

This "shadowing" only extends until the end of the dynamic
scope of the lambda above.
 
(Note: This code example assumes that `CREDENTIALS` is already bound
to a highly privileged set of credentials.)

### Inheritance

In our example above that uses a scope local to make credentials
available to callees, the callback is run on the same thread as the
caller. However, sometimes a method decomposes a task into several
parallel sub-tasks, but we still need to share some attributes with
them. For that we use scope-local inheritance.

Scope locals are either inheritable or non-inheritable. The
inheritability of a scope local is determined by its definition, like
so:

```
  static final ScopeLocal<MyType> x = ScopeLocal.inheritableForType(MyType.class);
```

Whenever `Thread` instances (virtual or not) are created, the set of
currently bound inheritable scope locals in the parent thread is
automatically inherited by each child thread:

```
    private static final ScopeLocal<Credentials> CREDENTIALS 
        = ScopeLocal.inheritableForType(Credentials.class);

    void multipleThreadExample() {
        Credentials creds = new Credentials("MySecret");
        ScopeLocal.where(CREDENTIALS, creds).run(() -> {
            for (int i = 0; i < 10; i++) {
                virtualThreadExecutor.submit(() -> {
                    // ...
                    Connection connection = connectDatabase();
                    // ...
                });
            }
        });
    }
```

(Implementation note: The scope-local credentials in this example are
not copied into each child thread. Instead, a reference to an
immutable set of scope-local bindings is passed to the child.)

Inheritable scope locals are also inherited by operations in a
parallel stream:

```
    void parallelStreamExample() {
        Credentials creds = new Credentials("MySecret");
        ScopeLocal.where(CREDENTIALS, creds).run(() -> {
            persons.parallelStream().forEach((Person p) -> connectDatabase().append(p));
        });
    }
```

In addition, a `Snapshot()` operation that captures the current set
of inheritable scope locals is provided. This allows context
information to be shared with asynchronous computations.

For example, a version of `ExecutorService.submit(Callable c)` that
captures the current set of inheritable scope locals looks like this:

```
    public <V> Future<V> submitWithCurrentSnapshot(ExecutorService executor, Callable<V> aCallable) {
        var s = ScopeLocal.snapshot();
        return executor.submit(() -> ScopeLocal.callWithSnapshot(aCallable, s));
    }
```

In this example, the `snapshot()` operation captures the state of
bound scope locals so that they can be retrieved by any code in the
dynamic scope of the `submit()` method, whichever thread it's run
on. These bound values are available for use even if the parent thread
(the one that invoked `submitWithCurrentSnapshot()`) has terminated.

### Optimization

Scope locals have some strongly defined properties. These can allow us
to generate excellent code for `get()`.

* The bound value of a scope local is effectively final within a
  method. It may be re-bound in a callee, but we know that when the
  callee terminates the scope local's value will have been
  restored. For that reason, we can hoist the value of a scope local
  into a register at the start of a method. Repeated uses of a scope
  local can be as fast as using a local variable.

* We don't have to check the type of a scope local every time we
  invoke `get()` because we know that its type was checked earlier
  when the scope local was bound.
  
### API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom-api/api/java.base/java/lang/ScopeLocal.html

## Alternatives

It is possible to emulate most of the features of scope locals with
`ThreadLocal`s, albeit at some cost in memory footprint, runtime
security, and performance. However, inheritable `ThreadLocal`s don't
really work with thread pools, and run the risk of leaking information
between unrelated tasks.

We have experimented with a modified version of `ThreadLocal` that
supports some of the characteristics of scope locals. However,
carrying the additional baggage of `ThreadLocal` results in an
implementation that is unduly burdensome, or an API that returns
`UnsupportedOperationException` for much core functionality, or
both. It is better, therefore, not to do that but to give scope locals
a separate identity from thread locals.
