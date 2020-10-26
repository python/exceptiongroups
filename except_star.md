### Disclaimer

* I'm going to be using the `ExceptionGroup` name in this issue, even
  though there are other alternatives, e.g. `AggregateException`. Naming of the
  "exception group" object is outside of the scope of this issue.

* This issue is primarily focused on discussing the new syntax modification
  proposal for the `try..except` construct, shortly called "except*".

* I use the term "naked" exception for regular Python exceptions
  **not wrapped** in an ExceptionGroup. E.g. a regular `ValueError`
  propagating through the stack is "naked".

* I assume that `ExceptionGroup` would be an iterable object.
  E.g. `list(ExceptionGroup(ValueError('a'), TypeError('b')))` would be
  equal to `[ValueError('a'), TypeError('b')]`

* I assume that `ExceptionGroup` won't be an indexable object; essentially
  it's similar to Python `set`. The motivation for this is that exceptions
  can occur in random order, and letting users write `group[0]` to access the
  "first" error is error prone. The actual implementation of `ExceptionGroup`
  will likely use an ordered list of errors though.

* I assume that `ExceptionGroup` will be a subclass of `BaseException`,
  which means it's assignable to `Exception.__context__` and can be
  directly handled with `except ExceptionGroup`.

* The behavior of good and old regular `try..except` will not be modified.

### Syntax

We're considering to introduce a new variant of the `try..except` syntax to
simplify working with exception groups:

```python
try:
  ...
except *SpamError:
  ...
except *BazError as e:
  ...
except *(BarError, FooError) as e:
  ...
```

The new syntax can be viewed as a variant of the tuple unpacking syntax.
The `*` symbol indicates that zero or more exceptions can be "caught" and
processed by one `except *` clause.

We also propose to enable "unpacking" in the `raise` statement:

```python
errors = (ValueError('hello'), TypeError('world'))
raise *errors
```

### Semantics

#### Overview

The  `except *SpamError` block will be run if the `try` code raised an
`ExceptionGroup` with one or more instances of `SpamError`. It would also be
triggered if a naked instance of  `SpamError` was raised.

The `except *BazError as e` block would aggregate all instances of `BazError`
into a list, wrap that list into an `ExceptionGroup` instance, and assign
the resultant object to `e`. The type of `e` would be
`ExceptionGroup[BazError]`.  If there was just one naked instance of
`BazError`, it would be wrapped into a list and assigned to `e`.

The `except *(BarError, FooError) as e` would aggregate all instances of
`BarError` or `FooError`  into a list and assign that wrapped list to `e`.
The type of `e` would be `ExceptionGroup[Union[BarError, FooError]]`.

Even though every `except*` star can be called only once, any number of
them can be run during handling of an `ExceptionGroup`. E.g. in the above
example,  both `except *SpamError:` and `except *(BarError, FooError) as e:`
could get executed during handling of one `ExceptionGroup` object, or all
of the `except*` clauses, or just one of them.

It is not allowed to use both regular `except` clauses and the new `except*`
clauses in the same `try` block. E.g. the following example would raise a
`SyntaxErorr`:

```python
try:
   ...
except ValueError:
   pass
except *CancelledError:
   pass
```

Exceptions are mached using a subclass check. For example:

```python
try:
  low_level_os_operation()
except *OSerror as errors:
  for e in errors:
    print(type(e).__name__)
```

could output:

```
BlockingIOError
ConnectionRefusedError
OSError
InterruptedError
BlockingIOError
```

#### New raise* Syntax

The new  `raise *` syntax allows to users to only process some exceptions
out of the matched set, e.g.:

```python
try:
  low_level_os_operation()
except *OSerror as errors:
  new_errors = []
  for e in errors:
    if e.errno != errno.EPIPE:
       new_errors.append(e)
  raise *new_errors
```

The above code ignores all `EPIPE` OS errors, while letting all others
propagate.

`raise *` syntax is special: it effectively extends the exception group with
a list of errors without creating a new `ExceptionGroup` instance:

```python
try:
  raise *(ValueError('a'), TypeError('b'))
except *ValueError:
  raise *(KeyError('x'), KeyError('y'))

# would result in:
#   ExceptionGroup({KeyError('x'), KeyError('y'), TypeError('b')})
```

A regular raise would behave similarly:

```python
try:
  raise *(ValueError('a'), TypeError('b'))
except *ValueError:
  raise KeyError('x')

# would result in:
#   ExceptionGroup({KeyError('x'), TypeError('b')})
```

`raise *` accepts arguments of type `Iterable[BaseException]`.

#### Unmatched Exceptions

Example:

```python
try:
  raise *(ValueError('a'), TypeError('b'), TypeError('c'), KeyError('e'))
except *ValueError as e:
  print(f'got some ValueErrors: {e}')
except *TypeError as e:
  print(f'got some TypeErrors: {e}')
  raise *e
```

The above code would print:

```
got some ValueErrors: ExceptionGroup({ValueError('a')})
got some TypeErrors: ExceptionGroup({TypeError('b'), TypeError('c')})
```

And then crash with an unhandled `KeyError('e')` error.

Basically, before interpreting `except *` clauses, the interpreter will
have an exception group object with a list of exceptions in it. Every
`except *` clause, evaluated from top to bottom, can filter some of the
exceptions out of the group and process them. In the end, if the exception
group has no exceptions left in it, it wold mean that all exceptions were
processed. If the exception group has some unprocessed exceptions, the current
frame will be "pushed" to the group's traceback and the group would be
propagated up the stack.

#### Exception Chaining

If an error occur during processing a set of exceptions in a `except *` block,
all matched errors would be put in a new `ExceptionGroup` which would have its
`__context__` attribute set to the just occurred exception:

```python
try:
  raise *(ValueError('a'), ValueError('b'), TypeError('z'))
except *ValueError:
  1 / 0

# would result in:
#
#   ExceptionGroup({
#     TypeError('z'),
#     ZeroDivisionError()
#   })
#
# where the `ZeroDivizionError()` instance would have
# its __context__ attribute set to
#
#   ExceptionGroup({
#     ValueError('a'), ValueError('b')
#   })
```

It's also possible to explicitly chain exceptions:

```python
try:
  raise *(ValueError('a'), ValueError('b'), TypeError('z'))
except *ValueError as errors:
  raise RuntimeError('unexpected values') from errors

# would result in:
#
#   ExceptionGroup(
#     TypeError('z'),
#     RuntimeError('unexpected values')
#   )
#
# where the `RuntimeError()` instance would have
# its __cause__ attribute set to
#
#   ExceptionGroup({
#     ValueError('a'), ValueError('b')
#   })
```

### Types of errors

Fundamentally there are two kinds of exceptions: *control flow exceptions*
and *operation exceptions*. The examples of former are `KeyboardInterrupt`,
`asyncio.CancelledError`, etc. Latter are `TypeError`, `KeyError`, etc.

When writing async/await code that uses a concept of TaskGroups (or Trio's
nurseries) to schedule different code concurrently, the users should
approach these kinds in a fundamentally different way.

*Operation exceptions* such as `KeyError` should be handled within
the async Task that runs the code.  E.g. this is what users should do:

```python
try:
  dct[key]
except KeyError:
  # handle the exception
```

and this is what they shouldn't do:

```python
try:
  async with asyncio.TaskGroup() as g:
    g.create_task(task1); g.create_task(task2)
except *KeyError:
  # handling KeyError here is meaningless, there's
  # no context to do anything with it but to log it.
```

*Control flow exceptions* are a different beast. If, for example, we want to
cancel an asyncio Task that spawned multiple concurrent Tasks in it with a
TaskGroup, we need to make sure that:

* CancelledErrors will be propagated to all still running tasks within
  the group

* CancelledErrors will be propagated to the Task that scheduled the group

* CancelledErrors will be propagated to the outer Tasks until either the entire
  program stops, or the exception is handled and silenced
  (e.g. by `asyncio.wait_for()`).

So suppose we have the `except *ExceptionType` syntax that can only handle
an ExceptionGroup with ExceptionType in it. This means that we'd see a lot
of code like this:


```python
try:
  async with asyncio.TaskGroup() as g:
    g.create_task(task1); g.create_task(task2)
except *CancelledError:
  log('cancelling server bootstrap')
  await server.stop()
  raise
except CancelledError:
  # Same code, really.
  log('cancelling server bootstrap')
  await server.stop()
  raise
```

Which led me to the conclusion that `except *CancelledError as e` should both:

* catch an individual standalone `CancelledError`, wrap it in an ExceptionGroup
  and bind it to `e`. The type of `e` is always
  `ExceptionGroup[CancelledError]`.

* if an exception group is propagating through the `try`,
  `except *CancelledError` should split the group and handle all exceptions
  at once with one run of the code in `except *CancelledError`.

Why "handle all exceptions at once with one run of the code code in except *"?
Why not run the code in the `except` clause for every matching exception that
we have in the group? Basically because there's no need to. As I mentioned
above, catching *operation exceptions* should be done with the regular old
`except KeyError` within the Task boundary, where there's context to handle
that `KeyError`. Catching *control flow exceptions* is needed to **react**
to some global signal, do cleanup or logging, but ultimately to either
**stop** the signal **or propagate** it up the caller chain.

Separating exceptions kinds to two distinct groups (operation & control flow)
leads to another conclusion: an individual `try..except` block usually handles
either the former or the latter, **but not a mix of both**. Which led me to
conclusion that `except *CancelledError` should switch the behavior of the
entire `try` block to make it run several of its `except *` clauses if
necessary.  So:

```python
try:
  # code
except KeyError:
  # handle
except ValueError:
  # handle
```

is the old and familiar `try..except`, we don't need to change it. And:

```python
try:
  # code
except *TimeoutError:
  # handle
except *CancelledError:
  # handle
```

is an entirely different mode and it's **OK**, and moreover, almost expected
from the user standpoint, to run both `except` clauses here.

And:

```python
try:
  # code
except ValueError:
  # handle
except *CancelledError:
  # handle
```

is weird and most likely suggests that the code should be refactored.

### Types of user code

Fundamentally we have applications and libraries. Applications are somewhat
simpler -- they typically can dictate what version of Python they require.
Which makes introducing TaskGroups and the new `except *` clause somewhat
easier. Basically, upon switching to Python 3.10, the application developer
can grep their application code for every *control flow* exception they handle
(search for `except CancelledError`) and mechanically change it to
`except *CancelledError`. Generally, judging by my own experience, there are
not so many places with code like that. Typically things like `CancelledError`
and `TimeoutError` are handled only in a few places, the rest of the code
relies on `try..finally` to cleanup its state.

Library developers are in a worse position: they'll need to maintain backwards
compatibility with older Pythons, so they can't start using the new `except *`
syntax. Two thoughts here:

This means that we'll need to have a proper programmatic API to work with
ExceptionGroups, so that libraries can write code like:

```python
try:
  # run user code
except CancelledError:
  # handle cancellation
except ExceptionGroup as e:
  g1, g2 = e.split(CancelledError)
  # handle cancellation
```

The API isn't going to be pretty, but that's OK, because a lot of existing
asyncio libraries don't run user-provided coroutines that might use TaskGroups.
In other words, a mysql or redis driver will never be able to catch an
ExceptionGroup until it starts using TaskGroups itself.

### Summary

I just cannot wrap my head around introducing `try..catch` syntax to Python.
I don't think we actually need it, with the right approach to documentation we
can explain to the users how to use TaskGroups and `except *` syntax correctly
and I believe it will only make their code better. Therefore,
to conclude I think we should:

* Introduce the ExceptionGroup object. Tweak the interpreter to recognize them
and do the correct thing to the wrapped Tracebacks etc.

* ExceptionGroups should have an API so that the few libraries that run user
code can maintain compatibility with older Pythons.

* `except *CancelledError as e` should catch:

  * an individual `CancelledError`; it would be wrapped in an ExceptionGroup
    created ad-hoc.
  * a single `CancelledError` wrapped in an ExceptionGroup
  * multiple `CancelledError`s at once.

* If a `try` block has two or more `except *` clauses it can run them one after
  another.

* Perhaps, if there's one `except *` in a `try` block, we should **require**
  all other `except` clauses to be written as `except *`.

* Handling exceptions like `*TimeoutError` is rather pointless. It's hard
  to correlate what specific subset of child tasks timed out -- the user
  should really handle timeouts in the sub-tasks where you have the context.
  So IMO exceptions groups are all about handling global signals like
  CancelledError or KeyboardInterrupt and about preserving the clear history
  of what crashed where and how to simplify development, debugging,
  and diagnostics.

### See Also

* An analysis of how exception groups will likely be used in asyncio
  programs:
  https://github.com/python/exceptiongroups/issues/3#issuecomment-716203284

* A WIP  implementation of the `ExceptionGroup` type by @iritkatriel
  tracked [here](https://github.com/iritkatriel/cpython/tree/exceptionGroup).


