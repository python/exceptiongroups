# Introducing try..except* syntax

## Disclaimer

* We use the `ExceptionGroup` name, even though there
  are other alternatives, e.g. `AggregateException` and `MultiError`.
  Naming of the "exception group" object is out of scope of this proposal.

* We use the term "naked" exception for regular Python exceptions
  **not wrapped** in an `ExceptionGroup`. E.g. a regular `ValueError`
  propagating through the stack is "naked".

* `ExceptionGroup` is an iterable object.
  E.g. `list(ExceptionGroup(ValueError('a'), TypeError('b')))` is
  equal to `[ValueError('a'), TypeError('b')]`

* `ExceptionGroup` is not an indexable object; essentially
  it's similar to Python `set`. The motivation for this is that exceptions
  can occur in random order, and letting users write `group[0]` to access the
  "first" error is error prone. Although the actual implementation of
  `ExceptionGroup` will likely use an ordered list of errors to preserve
  the actual occurrence order for rendering.

* `ExceptionGroup` is a subclass of `BaseException`,
  is assignable to `Exception.__context__`, and can be
  directly handled with `try: ... except ExceptionGroup: ...`.

* The behavior of the regular `try..except` statement will not be modified.

## Syntax

We're proposing to introduce a new variant of the `try..except` syntax to
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

## Semantics

### Overview

The `except *SpamError` block will be run if the `try` code raised an
`ExceptionGroup` with one or more instances of `SpamError`. It would also be
triggered if a naked instance of  `SpamError` was raised.

The `except *BazError as e` block would aggregate all instances of `BazError`
into a list, wrap that list into an `ExceptionGroup` instance, and assign
the resultant object to `e`. The type of `e` would be
`ExceptionGroup[BazError]`.  If there was just one naked instance of
`BazError`, it would be wrapped into an `ExceptionGroup` and assigned to `e`.

The `except *(BarError, FooError) as e` would aggregate all instances of
`BarError` or `FooError`  into a list and assign that wrapped list to `e`.
The type of `e` would be `ExceptionGroup[Union[BarError, FooError]]`.

Even though every `except*` clause can be executed only once, any number of
them can be run during handling of an `ExceptionGroup`. E.g. in the above
example,  both `except *SpamError:` and `except *(BarError, FooError) as e:`
could get executed during handling of one `ExceptionGroup` object, or all
of the `except*` clauses, or just one of them. However, each exception in
the exception group is only handled by one except* clause -- the first one
that matches its type.

It is not allowed to use both regular except blocks and the new `except*`
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

Exceptions are matched using a subclass check. For example:

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

### Unmatched Exceptions

Example:

```python
try:
  raise ExceptionGroup(
    ValueError('a'), TypeError('b'), TypeError('c'), KeyError('e')
  )
except *ValueError as e:
  print(f'got some ValueErrors: {e}')
except *TypeError as e:
  print(f'got some TypeErrors: {e}')
  raise
```

The above code would print:

```
got some ValueErrors: ExceptionGroup(ValueError('a'))
got some TypeErrors: ExceptionGroup(TypeError('b'), TypeError('c'))
```

and then terminate with an unhandled `ExceptionGroup`:

```
ExceptionGroup(
  TypeError('b'),
  TypeError('c'),
  KeyError('e'),
)
```

Basically, before interpreting `except *` clauses, the interpreter will
have an "incoming" `ExceptionGroup` object with a list of exceptions in it
to handle, and then:

* A new empty "result" `ExceptionGroup` would be created by the interpreter.

* Every `except *` clause, run from top to bottom, can match a subset of the
  exceptions out of the group forming a "working set" of errors for the current
  clause.  If the except block raises an exception, that exception is added
  to the "result" `ExceptionGroup` (with its "working set" of errors
  linked to that exception via the `__context__` attribute.)

* After there are no more `except*` clauses to evaluate, there are the
  following possibilities:

  * Both "incoming" and "result" `ExceptionGroup`s are empty. This means
    that all exceptions were processed and silenced.

  * Both "incoming" and "result" `ExceptionGroup`s are not empty.
    This means that not all of the exceptions were matched, and some were
    matched but either triggered new errors, or were re-raised. The interpreter
    would merge both groups into one group and raise it.

  * The "incoming" `ExceptionGroup` is non-empty: not all exceptions were
    processed. The interpreter would raise the "incoming" group.

  * The "result" `ExceptionGroup` is non-empty: all exceptions were processed,
    but some were re-raised or caused new errors. The interpreter would
    raise the "result" group.

The order of `except*` clauses is significant just like with the regular
`try..except`, e.g.:

```python
try:
  raise BlockingIOError
except *OSError as e:
  # Would catch the error
  print(e)
except *BlockingIOError:
  # Would never run
  print('never')

# would print:
#
#   ExceptionGroup(BlockingIOError())
```

### Raising ExceptionGroups manually

Exception groups can be raised manually:

```python
try:
  low_level_os_operation()
except *OSerror as errors:
  new_errors = []
  for e in errors:
    if e.errno != errno.EPIPE:
       new_errors.append(e)
  raise ExceptionGroup(*new_errors)
```

The above code ignores all `EPIPE` OS errors, while letting all other
exceptions propagate.

Raising an `ExceptionGroup` introduces nesting:

```python
try:
  raise ExceptionGroup(ValueError('a'), TypeError('b'))
except *ValueError:
  raise ExceptionGroup(KeyError('x'), KeyError('y'))

# would result in:
#
#   ExceptionGroup(
#     ExceptionGroup(
#       KeyError('x'),
#       KeyError('y'),
#     ),
#     TypeError('b'),
#   )
```

Although a regular `raise Exception` would not wrap `Exception` in a group:

```python
try:
  raise ExceptionGroup(ValueError('a'), TypeError('b'))
except *ValueError:
  raise KeyError('x')

# would result in:
#
#   ExceptionGroup(
#     KeyError('x'),
#     TypeError('b')
#   )
```

### Exception Chaining

If an error occurs during processing a set of exceptions in a `except *` block,
all matched errors would be put in a new `ExceptionGroup` which would be
referenced from the just occurred exception via its `__context__` attribute:

```python
try:
  raise ExceptionGroup(ValueError('a'), ValueError('b'), TypeError('z'))
except *ValueError:
  1 / 0

# would result in:
#
#   ExceptionGroup(
#     TypeError('z'),
#     ZeroDivisionError()
#   )
#
# where the `ZeroDivisionError()` instance would have
# its __context__ attribute set to
#
#   ExceptionGroup(
#     ValueError('a'),
#     ValueError('b')
#   )
```

It's also possible to explicitly chain exceptions:

```python
try:
  raise ExceptionGroup(ValueError('a'), ValueError('b'), TypeError('z'))
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
#   ExceptionGroup(
#     ValueError('a'),
#     ValueError('b')
#   )
```

### Recursive Matching

The matching of `except *` clauses against an `ExceptionGroup` is performed
recursively. E.g.:

```python
try:
  raise ExceptionGroup(
    ValueError('a'),
    TypeError('b'),
    ExceptionGroup(
      TypeError('c'),
      KeyError('d')
    )
  )
except *TypeError as e:
  print(f'got some TypeErrors: {list(e)}')
except *Exception:
  pass

# would print:
#
#  got some TypeErrors: [TypeError('b'), TypeError('c')]
```

Iteration over an `ExceptionGroup` that has nested `ExceptionGroup` objects
in it effectively flattens the entire tree. E.g.

```python
print(
  list(
    ExceptionGroup(
      ValueError('a'),
      TypeError('b'),
      ExceptionGroup(
        TypeError('c'),
        KeyError('d')
      )
    )
  )
)

# would output:
#
#   [ValueError('a'), TypeError('b'), TypeError('c'), KeyError('d')]
```

### Re-raising ExceptionGroups

It is important to point out that the `ExceptionGroup` bound to `e` is an
ephemeral object. Raising it via `raise` or `raise e` will not cause changes
to the overall shape of the `ExceptionGroup`.  Any modifications to it will
likely get lost:

```python
try:
  raise ExceptionGroup(
    ValueError('a'),
    TypeError('b'),
    ExceptionGroup(
      TypeError('c'),
      KeyError('d')
    )
  )
except *TypeError as e:
  e.foo = 'bar'
  # ^----------- `e` is an ephemeral object that might get
  #              destroyed after the `except*` clause.
```

If the user wants to "flatten" the tree, they can explicitly create a new
`ExceptionGroup` and raise it:


```python
try:
  raise ExceptionGroup(
    ValueError('a'),
    TypeError('b'),
    ExceptionGroup(
      TypeError('c'),
      KeyError('d')
    )
  )
except *TypeError as e:
  raise ExceptionGroup(*e)

# would terminate with:
#
#  ExceptionGroup(
#    ValueError('a'),
#    ExceptionGroup(
#      TypeError('b'),
#      TypeError('c'),
#    ),
#    ExceptionGroup(
#      KeyError('d')
#    )
#  )
```

With the regular exceptions, there's a subtle difference between bare `raise`
and a more specific `raise e`:

```python
def foo():                           | def foo():
  try:                               | try:
    1 / 0                            |   1 / 0
  except ZeroDivisionError as e:     | except ZeroDivisionError:
    raise e                          |   raise
                                     |
foo()                                | foo()
                                     |
Traceback (most recent call last):   | Traceback (most recent call last):
  File "/Users/guido/a.py", line 7   |   File "/Users/guido/b.py", line 7
    foo()                            |     foo()
  File "/Users/guido/a.py", line 5   |   File "/Users/guido/b.py", line 3
    raise e                          |     1/0
  File "/Users/guido/a.py", line 3   | ZeroDivisionError: division by zero
    1/0                              |
ZeroDivisionError: division by zero  |
```

This difference is preserved with exception groups:

* The `raise` form re-raises all exceptions from the group *without recording
  the current frame in their tracebacks*.

* The `raise e` form re-raises all exceptions from the group with tracebacks
  updated to point out to the current frame, effectively resulting in user
  seeing the `raise e` line in their tracebacks.

That said, both forms would not affect the overall shape of the exception
group:

```python
try:
  raise ExceptionGroup(
    ValueError('a'),
    TypeError('b'),
    ExceptionGroup(
      TypeError('c'),
      KeyError('d')
    )
  )
except *TypeError as e:
  raise  # or "raise e"

# would both terminate with:
#
#  ExceptionGroup(
#    ValueError('a'),
#    TypeError('b'),
#    ExceptionGroup(
#      TypeError('c'),
#      KeyError('d')
#    )
#  )
```

### "continue", "break", and "return" in "except*"

`continue`, `break`, and `return` are disallowed in `except*` clauses,
causing a `SyntaxError`.

Consider if they were allowed:

```python
def foo():
  try:
    raise ExceptionGroup(A(), B())
  except *A:
    return 1
  except *B:
    return 2

print(foo())
```

In the above example the user could guess that most likely the program
would print "1". But if instead of a simple `raise ExceptionGroup(A(), B())`
there's scheduling of a few concurrent tasks the answer is no longer obvious.

Ultimately though, due to the fact that a `try..except*` block allows multiple
`except*` clauses to run while handling one `ExceptionGroup` with
multiple different exceptions in it, allowing one innocent `break`, `continue`,
or `return` in one `except*` to effectively silence the entire group of
errors is error prone.

We can consider allowing some of them in future versions of Python.

## The Traceback of an Exception Group

For regular exceptions, the traceback represents a simple path of frames,
from the frame in which the exception was raised to the frame in which it was
was caught or, if it hasn't been caught yet, the frame that the program's
execution is currently in. The list is constructed by the interpreter which,
appends any frame it exits to the traceback of the 'current exception' if one
exists (the exception returned by `sys.exc_info()`). To support efficient appends,
the links in a traceback's list of frames are from the oldest to the newest frame.
Appending a new frame is then simply a matter of inserting a new head to the
linked list referenced from the exception's `__traceback__` field. Crucially,
the traceback's frame list is immutable in the sense that frames only need to
be added at the head, and never need to be removed.

We will not need to make any changes to this data structure. The
`__traceback__` field of the ExceptionGroup object represents that path that
the exceptions travelled through together after being joined into the
ExceptionGroup, and the same field on each of the nested exceptions represents
that path through which each exception arrived to the frame of the merge.

What we do need to change is any code that interprets and displays tracebacks,
because it will now need to continue into tracebacks of nested exceptions
once the traceback of an ExceptionGroup has been processed.

## Design Considerations

### Why try..except* syntax

Fundamentally there are two kinds of exceptions: *control flow exceptions*
(e.g. `KeyboardInterrupt` or `asyncio.CancelledError`) and
*operation exceptions* (e.g. `TypeError` or `KeyError`).

When writing async/await code that uses a concept of TaskGroups (or Trio's
nurseries) to schedule different code concurrently, the users should
approach these two kinds in a fundamentally different way.

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

*Control flow exceptions* are different. If, for example, we want to
cancel an asyncio Task that spawned other multiple concurrent Tasks in it
with a an `asyncio.TaskGroup`, the following will happen:

* CancelledErrors will be propagated to all still running tasks within
  the group;

* CancelledErrors will be propagated to the Task that scheduled the group and
  bubble up from `async with TaskGroup()`;

* CancelledErrors will be propagated to the outer Task until either the entire
  program shuts down with a `CancelledError`, or the cancellation is handled
  and silenced (e.g. by `asyncio.wait_for()`).

*Control flow exceptions* alter the execution flow of a program.
Therefore it is sometimes desirable for the user to react to them and
run code, for example, to free resources.

Suppose we have the `except *ExceptionType` syntax that only matches
`ExceptionGroup[ExceptionType]` exceptions (a naked `ExceptionType` wouldn't
be matched).  This means that we'd see a lot of code duplication:


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

Which leads to the conclusion that `except *CancelledError as e` should both:

* catch a naked `CancelledError`, wrap it in an `ExceptionGroup` and bind it
  to `e`. The type of `e` would always be `ExceptionGroup[CancelledError]`.

* if an exception group is propagating through the `try`,
  `except *CancelledError` should split the group and handle all exceptions
  at once with one run of the code in `except *CancelledError` (and not
  run the code for every matched individual exception.)

Why "handle all exceptions at once"? Why not run the code in the except
clause for every matched exception that we have in the group?
Basically because there's no need to. As we mentioned above, catching
*operation exceptions* should be done with the regular `except KeyError`
within the Task boundary, where there's context to handle a `KeyError`.
Catching *control flow exceptions* is needed to **react** to a global
signal, do cleanup or logging, but ultimately to either **stop** the signal
**or propagate** it up the caller chain.

Separating exception kinds to two distinct groups (operation & control flow)
leads to another conclusion: an individual `try..except` block usually handles
either the former or the latter, **but not a mix of both**. Which leads to the
conclusion that `except *CancelledError` should switch the behavior of the
entire `try` block to make it run several of its `except *` clauses if
necessary.  Therefore:

```python
try:
  # code
except KeyError:
  # handle
except ValueError:
  # handle
```

is a regular `try..except` block to be used for reacting to
*operation exceptions*. And:

```python
try:
  # code
except *TimeoutError:
  # handle
except *CancelledError:
  # handle
```

is an entirely different construct meant to make it easier to react to
*control flow* signals.  When specified that way, it is expected from the user
standpoint that both `except` clauses can be potentially run.

Lastly, code that combines handling of both operation and control flow
exceptions is unrealistic and impractical, e.g.:

```python
try:
  async with TaskGroup() as g:
    g.create_task(task1())
    g.create_task(task2())
except ValueError:
  # handle ValueError
except *CancelledError:
  # handle cancellation
  raise
```

In the above snippet it is impossible to attribute which task raised a
`ValueError` -- `task1` or `task2`. So it really should be handled directly
in those tasks. Whereas handling `*CancelledError` makes sense -- it means that
the current task is being canceled and this might be a good opportunity to do
a cleanup.


### Adoption of try..except* syntax

Application code typically can dictate what version of Python it requires.
Which makes introducing TaskGroups and the new `except *` clause somewhat
straightforward. Upon switching to Python 3.10, the application developer
can grep their application code for every *control flow* exception they handle
(search for `except CancelledError`) and mechanically change it to
`except *CancelledError`.

Library developers, on the other hand, will need to maintain backwards
compatibility with older Python versions, and therefore they wouldn't be able
to start using the new `except *` syntax right away.  They will have to use
the new ExceptionGroup low-level APIs along with `try..except ExceptionGroup`
to support running user code that can raise exception groups.

### Traceback Representation

We considered options for adapting the traceback data structure to represent
trees, but it became apparent that a traceback tree is not meaningful once separated
from the exceptions it refers to. While a simple-path traceback can be attached to
any exception by a `with_traceback()` call, it is hard to imagine a case where it
makes sense to assign a traceback tree to an exception group.  Furthermore, a
useful display of the traceback includes information about the nested exceptions.
For this reason we decided it is best to leave the traceback mechanism as it is
and modify the traceback display code.

## See Also

* An analysis of how exception groups will likely be used in asyncio
  programs:
  https://github.com/python/exceptiongroups/issues/3#issuecomment-716203284

* A WIP implementation of the `ExceptionGroup` type by @iritkatriel
  tracked [here](https://github.com/iritkatriel/cpython/tree/exceptionGroup).

* The issue where this concept was first formalized:
  https://github.com/python/exceptiongroups/issues/4
