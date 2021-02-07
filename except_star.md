# Introducing try..except* syntax


## Abstract

This PEP proposes language extensions that allow programs to raise and handle
multiple unrelated exceptions simultaneously:

* A new standard exception type, the `ExceptionGroup`, which represents a
group of unrelated exceptions being propagated together.

* A new syntax `except*` for handling `ExceptionGroup`s.

## Motivation

The interpreter is currently able to propagate at most one exception at a
time. The chaining features introduced in
[PEP 3134](https://www.python.org/dev/peps/pep-3134/) link together exceptions that are
related to each other as the cause or context, but there are situations where
multiple unrelated exceptions need to be propagated together as the stack
unwinds. Several real world use cases are listed below.

* **Concurrent errors**. Libraries for async concurrency provide APIs to invoke
  multiple tasks and return their results in aggregate. There isn't currently
  a good way for such libraries to handle situations where multiple tasks
  raise exceptions. The Python standard library's
  [`asyncio.gather()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)
  function provides two options: raise the first exception, or return the
  exceptions in the results list.  The [Trio](https://trio.readthedocs.io/en/stable/)
  library has a `MultiError` exception type which it raises to report a
  collection of errors. Work on this PEP was initially motivated by the
  difficulties in handling `MultiError`s, which are detailed in a design
  document for an
  [improved version, `MultiError2`]([https://github.com/python-trio/trio/issues/611).
  That document demonstrates how difficult it is to create an effective API
  for reporting and handling multiple errors without the language changes we
  are proposing.

* **Multiple failures when retrying an operation.** The Python standard library's
  `socket.create_connection` may attempt to connect to different addresses,
  and if all attempts fail it needs to report that to the user. It is an open
  issue how to aggregate these errors, particularly when they are different
  [[Python issue 29980](https://bugs.python.org/issue29980)].

* **Multiple user callbacks fail.** The pytest library allows users to register
  finalizers which are executed at teardown. If more than one of these
  finalizers raises an exception, only the first is reported to the user. This
  can be improved with `ExceptionGroup`s, as explained in this issue by pytest
  developer Ran Benita [[Pytest issue 8217](https://github.com/pytest-dev/pytest/issues/8217)]

* **Multiple errors in a complex calculation.** The Hypothesis library performs
  automatic bug reduction (simplifying code that demonstrates a bug). In the
  process it may find variations that generate different errors, and
  (optionally) reports all of them
  [[Hypothesis documentation](https://hypothesis.readthedocs.io/en/latest/settings.html#hypothesis.settings.report_multiple_bugs)].
  An `ExceptionGroup` mechanism as we are proposing here can resolve some of
  the difficulties with debugging that are mentioned in the link above, and
  which are due to the loss of context/cause information (communicated
  by Hypothesis Core Developer Zac Hatfield-Dodds).

* **Errors in wrapper code.** The Python standard library's
  `tempfile.TemporaryDirectory` context manager
  had an issue where an exception raised during cleanup in `__exit__`
  effectively masked an exception that the user's code raised inside the context
  manager scope. While the user's exception was chained as the context of the
  cleanup error, it was not caught by the user's except clause
  [[Python issue 40857](https://bugs.python.org/issue40857)].
  The issue was resolved by making the cleanup code ignore errors, thus
  sidestepping the multiple exception problem. With the features we propose
  here, it would be possible for `__exit__` to raise an `ExceptionGroup`
  containing its own errors as well as the user's errors as unrelated errors,
  and this would allow the user to catch their own exceptions by their types.


## Rationale

Grouping several exceptions together can be done without changes to the
language, simply by creating a container exception type. Trio is an example of
a library that has made use of this technique in its `MultiError` type
[reference to Trio MultiError]. However, such approaches require calling code
to catch the container exception type, and then inspect it to determine the
types of errors that had occurred, extract the ones it wants to handle and
reraise the rest.

Changes to the language are required in order to extend support for
`ExceptionGroup`s in the style of existing exception handling mechanisms. At
the very least we would like to be able to catch an `ExceptionGroup` only if
it contains an exception type that we that chose to handle. Exceptions of
other types in the same `ExceptionGroup` need to be automatically reraised,
otherwise it is too easy for user code to inadvertently swallow exceptions
that it is not handling.

The purpose of this PEP, then, is to add the `except*` syntax for handling
`ExceptionGroups`s in the interpreter, which in turn requires that
`ExceptionGroup` is added as a builtin type. The semantics of handling
`ExceptionGroup`s are not backwards compatible with the current exception
handling semantics, so we could not modify the behaviour of the `except`
keyword and instead added the new `except*` syntax.


## Specification

### ExceptionGroup

The new builtin exception type, `ExceptionGroup` is a subclass of
`BaseException`, so it is assignable to `Exception.__cause__` and
`Exception.__context__`, and can be raised and handled as any exception
with `raise ExceptionGroup(...)` and `try: ... except ExceptionGroup: ...`.

Its constructor takes two positional-only parameters: a message string and a
sequence of the nested exceptions, for example:
`ExceptionGroup('issues', [ValueError('bad value'), TypeError('bad type')])`.

The `ExceptionGroup` class exposes these parameters in the fields `message`
and `errors`.  A nested exception can also be an `ExceptionGroup` so the class
represents a tree of exceptions, where the leaves are plain exceptions and
each internal node represent a time at which the program grouped some
unrelated exceptions into a new `ExceptionGroup`.

The `ExceptionGroup.subgroup(condition)` method gives us a way to obtain an
`ExceptionGroup` that has the same metadata (cause, context, traceback) as
the original group, and the same nested structure of `ExceptionGroup`s, but
contains only those exceptions for which the condition is true:

```python
>>> eg = ExceptionGroup("one",
...                     [TypeError(1),
...                      ExceptionGroup("two",
...                                     [TypeError(2), ValueError(3)]),
...                      ExceptionGroup("three",
...                                     [OSError(4)])
...                     ])
>>> traceback.print_exception(eg)
ExceptionGroup: one
   ------------------------------------------------------------
   TypeError: 1
   ------------------------------------------------------------
   ExceptionGroup: two
      ------------------------------------------------------------
      TypeError: 2
      ------------------------------------------------------------
      ValueError: 3
   ------------------------------------------------------------
   ExceptionGroup: three
      ------------------------------------------------------------
      OSError: 4
>>> type_errors = eg.subgroup(lambda e: isinstance(e, TypeError))
>>> traceback.print_exception(type_errors)
ExceptionGroup: one
   ------------------------------------------------------------
   TypeError: 1
   ------------------------------------------------------------
   ExceptionGroup: two
      ------------------------------------------------------------
      TypeError: 2
>>>
```

Empty nested `ExceptionGroup`s are omitted from the result, as in the
case of `ExceptionGroup("three")` in the example above.  The original `eg`
is unchanged by `subgroup`, but the value returned is not necessarily a full
new copy. Leaf exceptions are not copied, nor are `ExceptionGroup`s which are
fully contained in the result. When it is necessary to partition an
`ExceptionGroup` because the condition holds for some, but not all of its
contained exceptions, a new `ExceptionGroup` is created but the `__cause__`,
`__context__` and `__traceback__` field are copied by reference, so are shared
with the original `eg`.

If both the subgroup and its complement are needed, the `ExceptionGroup.split`
method can be used:

```Python
>>> type_errors, other_errors = eg.split(lambda e: isinstance(e, TypeError))
>>> traceback.print_exception(type_errors)
ExceptionGroup: one
   ------------------------------------------------------------
   TypeError: 1
   ------------------------------------------------------------
   ExceptionGroup: two
      ------------------------------------------------------------
      TypeError: 2
>>> traceback.print_exception(other_errors)
ExceptionGroup: one
   ------------------------------------------------------------
   ExceptionGroup: two
      ------------------------------------------------------------
      ValueError: 3
   ------------------------------------------------------------
   ExceptionGroup: three
      ------------------------------------------------------------
      OSError: 4
>>>
```

If a split is trivial (one side is empty), then None is returned for the
other side:

```python
>>> other_errors.split(lambda e: isinstance(e, SyntaxError))
(None, ExceptionGroup('one', [ExceptionGroup('two', [ValueError(3)]), ExceptionGroup('three', [OSError(4)])]))
```

Since splitting by exception type is a very common use case, `subgroup` and
`split` can take an exception type or tuple of exception types and treat it
as a shorthand for matching that type: `eg.split(TypeError)`, is equivalent to
`eg.split(lambda e: isinstance(e, TypeError))`.


#### The Traceback of an `ExceptionGroup`

For regular exceptions, the traceback represents a simple path of frames,
from the frame in which the exception was raised to the frame in which it
was caught or, if it hasn't been caught yet, the frame that the program's
execution is currently in. The list is constructed by the interpreter, which
appends any frame from which it exits to the traceback of the 'current
exception' if one exists. To support efficient appends, the links in a
traceback's list of frames are from the oldest to the newest frame. Appending
a new frame is then simply a matter of inserting a new head to the linked
list referenced from the exception's `__traceback__` field. Crucially, the
traceback's frame list is immutable in the sense that frames only need to be
added at the head, and never need to be removed.

We do not need to make any changes to this data structure. The `__traceback__`
field of the `ExceptionGroup` instance represents the path that the contained
exceptions travelled through together after being joined into the
`ExceptionGroup`, and the same field on each of the nested exceptions
represents the path through which this exception arrived at the frame of the
merge.

What we do need to change is any code that interprets and displays tracebacks,
because it now needs to continue into tracebacks of nested exceptions, as
in the following example:

```python
>>> def f(v):
...     try:
...         raise ValueError(v)
...     except ValueError as e:
...         return e
...
>>> try:
...     raise ExceptionGroup("one", [f(1)])
... except ExceptionGroup as e:
...     eg1 = e
...
>>> try:
...     raise ExceptionGroup("two", [f(2), eg1])
... except ExceptionGroup as e:
...     eg2 = e
...
>>> import traceback
>>> traceback.print_exception(eg2)
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
ExceptionGroup: two
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in f
   ValueError: 2
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 2, in <module>
   ExceptionGroup: one
      ------------------------------------------------------------
      Traceback (most recent call last):
        File "<stdin>", line 3, in f
      ValueError: 1
>>>
```

### except*

We're proposing to introduce a new variant of the `try..except` syntax to
simplify working with exception groups. The `*` symbol indicates that multiple
exceptions can be handled by each `except*` clause:

```python
try:
  ...
except *SpamError:
  ...
except *FooError as e:
  ...
except *(BarError, BazError) as e:
  ...
```

In a traditional `try-except` statement there is only one exception to handle,
so the body of at most one `except` clause executes; the first one that matches
the exception. With the new syntax, an `except*` clause can match a subgroup
of the `ExceptionGroup` that was raised, while the remaining part is matched
by following `except*` clauses. In other words, a single `ExceptionGroup` can
cause several `except*` clauses to execute, but each such clause executes at
most once (for all matching exceptions from the group) and each exception is
either handled by exactly one clause (the first one that matches its type)
or is reraised at the end.

For example, suppose that the body of the `try` block above raises
`eg = ExceptionGroup('msg', [FooError(1), FooError(2), BazError()])`.
The `except*` clauses are evaluated in order by calling `split` on the
`unhandled` `ExceptionGroup`, which is initially equal to `eg` and then shrinks
as exceptions are matched and extracted from it.

In our example, `unhandled.split(SpamError)` returns `(None, unhandled)` so the
first `except*` block is not executed and `unhandled` is unchanged. For the
second block, `match, rest = unhandled.split(FooError)` returns a non-trivial
split with `match = ExceptionGroup('msg', [FooError(1), FooError(2)])`
and `rest = ExceptionGroup('msg', [BazError()])`. The body of this `except*`
block is executed, with the value of `e` and `sys.exc_info()` set to `match`.
Then, `unhandled` is set to `rest`.
Finally, the third block matches the remaining exception so it is executed
with `e` and `sys.exc_info()` set to `ExceptionGroup('msg', [BazError()])`.


Exceptions are matched using a subclass check. For example:

```python
try:
  low_level_os_operation()
except *OSerror as eg:
  for e in eg.errors:
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

The order of `except*` clauses is significant just like with the regular
`try..except`:

```python
>>> try:
...    raise ExceptionGroup("problem", [BlockingIOError()])
... except *OSError as e:   # Would catch the error
...   print(repr(e))
... except *BlockingIOError: # Would never run
...   print('never')
...
ExceptionGroup('problem', [BlockingIOError()])
```

### Recursive Matching

The matching of `except*` clauses against an `ExceptionGroup` is performed
recursively, using the `ExceptionGroup.split()` method:

```python
>>> try:
...   raise ExceptionGroup(
...     "eg",
...     [ValueError('a'),
...      TypeError('b'),
...      ExceptionGroup("nested", [TypeError('c'), KeyError('d')])
...     ]
...   )
... except *TypeError as e1:
...   print(f'e1 = {e1!r}')
... except *Exception as e2:
...   print(f'e2 = {e2!r}')
...
e1 = ExceptionGroup('eg', [TypeError('b'), ExceptionGroup('nested', [TypeError('c')])])
e2 = ExceptionGroup('eg', [ValueError('a'), ExceptionGroup('nested', [KeyError('d')])])
>>>
```

### Unmatched Exceptions

If not all exceptions in an `ExceptionGroup` were matched by the `except*`
clauses, the remaining part of the `ExceptionGroup` is propagated on:

```python
>>> try:
...   try:
...     raise ExceptionGroup(
...       "msg", [ValueError('a'), TypeError('b'), TypeError('c'), KeyError('e')]
...     )
...   except *ValueError as e:
...     print(f'got some ValueErrors: {e!r}')
...   except *TypeError as e:
...     print(f'got some TypeErrors: {e!r}')
... except ExceptionGroup as e:
...   print(f'propagated: {e!r}')
...
got some ValueErrors: ExceptionGroup('msg', [ValueError('a')])
got some TypeErrors: ExceptionGroup('msg', [TypeError('b'), TypeError('c')])
propagated: ExceptionGroup('msg', [KeyError('e')])
>>>
```

### Naked Exceptions

If the exception raised inside the `try` body is not of type `ExceptionGroup`,
we call it a `naked` exception. If its type matches one of the `except*`
clauses, it is wrapped by an `ExceptionGroup` with an empty message string
when caught. This is to make the type of `e` consistent and statically known:

```python
>>> try:
...    raise BlockingIOError
... except *OSError as e:
...    print(repr(e))
...
ExceptionGroup('', [BlockingIOError()])
```

However, if a naked exception is not caught, it propagates in its original
naked form:

```python
>>> try:
...   try:
...     raise ValueError(12)
...   except *TypeError as e:
...     print('never')
... except ValueError as e:
...   print(f'caught ValueError: {e!r}')
...
caught ValueError: ValueError(12)
>>>
```

### Raising exceptions in an `except*` block

In a traditional `except` block, there are two ways to raise exceptions:
`raise e` to explicitly raise an exception object `e`, or naked `raise` to
reraise the 'current exception'. When `e` is the current exception, the two
forms are not equivalent because a reraise does not add the current frame to
the stack:

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

This holds for `ExceptionGroup`s as well, but the situation is now more complex
because there can exceptions raised and reraised from multiple `except*`
clauses, as well as unhandled exceptions that need to propagate.
The interpreter needs to combine all those exceptions into a result, and
raise that.

The reraised exceptions and the unhandled exceptions are subgroups of the
original `ExceptionGroup`, and share its metadata (cause, context, traceback).
On the other hand, each of the explicitly raised exceptions has its own
metadata - the traceback contains the line from which it was raised, its
cause is whatever it may have been explicitly chained to, and its context is the
value of `sys.exc_info()` in the `except*` clause of the raise.

In the aggregated `ExceptionGroup`,  the reraised and unhandled exceptions have
the same relative structure as in the original exception, as if they were split
off  together in one `subgroup` call. For example, in the snippet below the
inner `try-except*` block raises an `ExceptionGroup` that contains all
`ValueError`s and `TypeError`s merged back into the same shape they had in
the original `ExceptionGroup`:

```python
>>> try:
...   try:
...     raise ExceptionGroup("eg",
...                           [ValueError(1),
...                           TypeError(2),
...                           OSError(3),
...                           ExceptionGroup(
...                              "nested",
...                              [OSError(4), TypeError(5), ValueError(6)])])
...   except *ValueError as e:
...     print(f'*ValueError: {e!r}')
...     raise
...   except *OSError as e:
...     print(f'*OsError: {e!r}')
... except ExceptionGroup as e:
...   print(repr(e))
...
*ValueError: ExceptionGroup('eg', [ValueError(1), ExceptionGroup('nested', [ValueError(6)])])
*OsError: ExceptionGroup('eg', [OSError(3), ExceptionGroup('nested', [OSError(4)])])
ExceptionGroup('eg', [ValueError(1), TypeError(2), ExceptionGroup('nested', [TypeError(5), ValueError(6)])])
>>>
```

When exceptions are raised explicitly, they are independent of the original
exception group, and cannot be merged with it (they have their own cause,
context and traceback). Instead, they are combined into a new `ExceptionGroup`,
which also contains the reraised/unhandled subgroup described above.

In the following example, the `ValueError`s were raised so they are in their
own `ExceptionGroup`, while the `OSError`s were reraised so they were
merged with the unhandled `TypeError`s.

```python
>>> try:
...    try:
...      raise ExceptionGroup("eg",
...                            [ValueError(1),
...                            TypeError(2),
...                            OSError(3),
...                            ExceptionGroup(
...                               "nested",
...                               [OSError(4), TypeError(5), ValueError(6)])])
...    except *ValueError as e:
...      print(f'*ValueError: {e!r}')
...      raise e
...    except *OSError as e:
...      print(f'*OSError: {e!r}')
...      raise
... except ExceptionGroup as e:
...    traceback.print_exception(e)
...
*ValueError: ExceptionGroup('eg', [ValueError(1), ExceptionGroup('nested', [ValueError(6)])])
*OSError: ExceptionGroup('eg', [OSError(3), ExceptionGroup('nested', [OSError(4)])])
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ExceptionGroup
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 12, in <module>
     File "<stdin>", line 3, in <module>
   ExceptionGroup: eg
      ------------------------------------------------------------
      ValueError: 1
      ------------------------------------------------------------
      ExceptionGroup: nested
         ------------------------------------------------------------
         ValueError: 6
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: eg
      ------------------------------------------------------------
      TypeError: 2
      ------------------------------------------------------------
      OSError: 3
      ------------------------------------------------------------
      ExceptionGroup: nested
         ------------------------------------------------------------
         OSError: 4
         ------------------------------------------------------------
         TypeError: 5
>>>
```

### Chaining

Explicitly raised `ExceptionGroup`s are chained as with any exceptions. The
following example shows how part of `ExceptionGroup` "one" became the
context for `ExceptionGroup` "two", while the other part was combined with
it into the new `ExceptionGroup`.

```python
>>> try:
...    try:
...      raise ExceptionGroup("one", [ValueError('a'), TypeError('b')])
...    except *ValueError:
...      raise ExceptionGroup("two", [KeyError('x'), KeyError('y')])
... except BaseException as e:
...    traceback.print_exception(e)
...
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ExceptionGroup
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: one
      ------------------------------------------------------------
      ValueError: a

   During handling of the above exception, another exception occurred:

   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
   ExceptionGroup: two
      ------------------------------------------------------------
      KeyError: 'x'
      ------------------------------------------------------------
      KeyError: 'y'

   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: one
      ------------------------------------------------------------
      TypeError: b
```

### Raising New Exceptions

In the previous examples the explicit raises were of the exceptions that
were caught, so for completion we show a new exception being raise, with
chaining:

```python
>>> try:
...   try:
...     raise TypeError('bad type')
...   except *TypeError as e:
...     raise ValueError('bad value') from e
... except ExceptionGroup as e:
...   traceback.print_exception(e)
...
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ExceptionGroup
   ------------------------------------------------------------
   ExceptionGroup
      ------------------------------------------------------------
      Traceback (most recent call last):
        File "<stdin>", line 3, in <module>
      TypeError: bad type

   The above exception was the direct cause of the following exception:

   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
   ValueError: bad value
>>>
```

Note that exceptions raised in one `except*` clause are not eligible to match
other clauses from the same `try` statement:

```python
>>> try:
...   try:
...     raise TypeError(1)
...   except *TypeError:
...     raise ValueError(2)   # <- not caught in the next clause
...   except *ValueError:
...     print('never')
... except ExceptionGroup as e:
...   traceback.print_exception(e)
...
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ExceptionGroup
   ------------------------------------------------------------
   ExceptionGroup
      ------------------------------------------------------------
      Traceback (most recent call last):
        File "<stdin>", line 3, in <module>
      TypeError: 1

   During handling of the above exception, another exception occurred:

   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
   ValueError: 2
```


Raising a new instance of a naked exception does not cause this exception to
be wrapped by an `ExceptionGroup`. Rather, the exception is raised as is, and
if it needs to be combined with other propagated exceptions, it becomes a
direct child of the new `ExceptionGroup` created for that:


```python
>>> try:
...   try:
...     raise ExceptionGroup("eg", [ValueError('a')])
...   except *ValueError:
...     raise KeyError('x')
... except BaseException as e:
...     traceback.print_exception(e)
...
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ExceptionGroup
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: eg
      ------------------------------------------------------------
      ValueError: a

   During handling of the above exception, another exception occurred:

   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
   KeyError: 'x'
>>>
>>> try:
...   try:
...     raise ExceptionGroup("eg", [ValueError('a'), TypeError('b')])
...   except *ValueError:
...     raise KeyError('x')
... except BaseException as e:
...     traceback.print_exception(e)
...
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ExceptionGroup
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: eg
      ------------------------------------------------------------
      ValueError: a

   During handling of the above exception, another exception occurred:

   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
   KeyError: 'x'

   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: eg
      ------------------------------------------------------------
      TypeError: b
>>>
```

Finally, as an example of how the proposed API can help us work effectively
with `ExceptionGroup`s, the following code ignores all `EPIPE` OS errors,
while letting all other exceptions propagate.

```python
try:
  low_level_os_operation()
except *OSerror as errors:
  raise errors.subgroup(lambda e: e.errno != errno.EPIPE) from None
```

### Caught Exception Objects

It is important to point out that the `ExceptionGroup` bound to `e` is an
ephemeral object. Raising it via `raise` or `raise e` will not cause changes
to the overall shape of the `ExceptionGroup`.  Any modifications to it will
likely get lost:

```python
>>> eg = ExceptionGroup("eg", [TypeError(12)])
>>> eg.foo = 'foo'
>>> try:
...   raise eg
... except *TypeError as e:
...   e.foo = 'bar'
... # ^----------- `e` is an ephemeral object that might get
>>> #              destroyed after the `except*` clause.
>>> eg.foo
'foo'
>>>
```

### Forbidden Combinations

* It is not possible to use both regular `except` blocks and the new `except*`
clauses in the same `try` statement.The following example would raise a
`SyntaxErorr`:

```python
try:
   ...
except ValueError:
   pass
except *CancelledError:  # <- SyntaxError:
   pass                  #    combining `except` and `except*` is prohibited
```

* It is possible to catch the `ExceptionGroup` type with a plain except, but not
with an `except*` because the latter is ambiguous:

```python
try:
   ...
except ExceptionGroup:  # <- This works
   pass

try:
   ...
except *ExceptionGroup:  # <- Runtime error
   pass
```

* An empty "match anything" `except*` block is not supported as its meaning may
be confusing:

```python
try:
   ...
except*:   # <- SyntaxError
  pass
```

* `continue`, `break`, and `return` are disallowed in `except*` clauses,
causing a `SyntaxError`.

This is because the exceptions in an `ExceptionGroup` are assumed to be
independent, and the presence or absence of one of them should not impact
handling of the others, as could happen if we allow an `except*` clause to
change the way control flows through other clauses.  We believe that this is
error prone and there are better ways to implement a check like this:

```python
def foo():
  try:
    raise ExceptionGroup("msg", A(), B())
  except *A:
    return 1   # <- SyntaxError
  except *B as e:
    raise TypeError("Can't have B without A!") from e
```

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
entire `try` block to make it run several of its `except*` clauses if
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

## Backwards Compatibility

The behaviour of `except` is unchanged so existing code will continue to work.

### Adoption of try..except* syntax

Application code typically can dictate what version of Python it requires.
Which makes introducing TaskGroups and the new `except*` clause somewhat
straightforward. Upon switching to Python 3.10, the application developer
can grep their application code for every *control flow* exception they handle
(search for `except CancelledError`) and mechanically change it to
`except *CancelledError`.

Library developers, on the other hand, will need to maintain backwards
compatibility with older Python versions, and therefore they wouldn't be able
to start using the new `except*` syntax right away.  They will have to use
the new ExceptionGroup low-level APIs along with `try..except ExceptionGroup`
to support running user code that can raise exception groups.


## Security Implications

## How to Teach This

## Reference Implementation

[An experimental implementation](https://github.com/iritkatriel/cpython/tree/exceptionGroup-stage5).


## Rejected Ideas

### The ExceptionGroup API

We considered making `ExceptionGroup`s iterable, so that `list(eg)` would
produce a flattened list of the plain exceptions contained in the group.
We decided that this would not be not be a sound API, because the metadata
(cause, context and traceback) of the individual exceptions in a group are
incomplete and this could create problems.  If use cases arise where this
can be helpful, we can document (or even provide in the standard library)
a sound recipe for accessing an individual exception: use the `split()`
method to create an `ExceptionGroup` for a single exception and then
transform it into a plain exception with the current metadata.

### Traceback Representation

We considered options for adapting the traceback data structure to represent
trees, but it became apparent that a traceback tree is not meaningful once
separated from the exceptions it refers to. While a simple-path traceback can
be attached to any exception by a `with_traceback()` call, it is hard to
imagine a case where it makes sense to assign a traceback tree to an exception
group.  Furthermore, a useful display of the traceback includes information
about the nested exceptions. For this reason we decided it is best to leave
the traceback mechanism as it is and modify the traceback display code.

### A full redesign of `except`

We considered introducing a new keyword (such as `catch`) which can be used
to handle both plain exceptions and `ExceptionGroup`s. Its semantics would
be the same as those of `except*` when catching an `ExceptionGroup`, but
it would not wrap plain a exception to create an `ExceptionGroup`. This
would have been part of a long term plan to replace `except` by `catch`,
but we decided that deprecating `except` in favour of an enhanced keyword
would be too confusing for users at this time, so it is more appropriate
to introduce the `except*` syntax for `ExceptionGroup`s while `except`
continues to be used for simple exceptions.


## See Also

* An analysis of how exception groups will likely be used in asyncio
  programs:
  https://github.com/python/exceptiongroups/issues/3#issuecomment-716203284

  * The issue where this concept was first formalized:
  https://github.com/python/exceptiongroups/issues/4


## References

## Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
