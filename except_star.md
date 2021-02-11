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
  [improved version, `MultiError2`](https://github.com/python-trio/trio/issues/611).
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
language, simply by creating a container exception type.
[Trio](https://trio.readthedocs.io/en/stable/) is an example of a library that
has made use of this technique in its
[`MultiError` type](https://trio.readthedocs.io/en/stable/reference-core.html#trio.MultiError).
However, such approaches require calling code to catch the container exception
type, and then inspect it to determine the types of errors that had occurred,
extract the ones it wants to handle and reraise the rest.

Changes to the language are required in order to extend support for
`ExceptionGroup`s in the style of existing exception handling mechanisms. At
the very least we would like to be able to catch an `ExceptionGroup` only if
it contains an exception type that we choose to handle. Exceptions of
other types in the same `ExceptionGroup` need to be automatically reraised,
otherwise it is too easy for user code to inadvertently swallow exceptions
that it is not handling.

The purpose of this PEP, then, is to add the `except*` syntax for handling
`ExceptionGroup`s in the interpreter, which in turn requires that
`ExceptionGroup` is added as a builtin type. The semantics of handling
`ExceptionGroup`s are not backwards compatible with the current exception
handling semantics, so we are not proposing to modify the behavior of the
`except` keyword but rather to add the new `except*` syntax.


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
unrelated exceptions into a new `ExceptionGroup` and raised them together.
The `ExceptionGroup` class is final, i.e., it cannot be subclassed.

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
`__context__` and `__traceback__` fields are copied by reference, so are shared
with the original `eg`.

If both the subgroup and its complement are needed, the
`ExceptionGroup.split(condition)` method can be used:

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

We are proposing to introduce a new variant of the `try..except` syntax to
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
as exceptions are matched and extracted from it.  In the first `except*` clause,
`unhandled.split(SpamError)` returns `(None, unhandled)` so the body of this
block is not executed and `unhandled` is unchanged. For the second block,
`unhandled.split(FooError)` returns a non-trivial split `(match, rest)` with
`match = ExceptionGroup('msg', [FooError(1), FooError(2)])`
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
clauses, it is caught and wrapped by an `ExceptionGroup` with an empty message
string. This is to make the type of `e` consistent and statically known:

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
because there can be exceptions raised and reraised from multiple `except*`
clauses, as well as unhandled exceptions that need to propagate.
The interpreter needs to combine all those exceptions into a result, and
raise that.

The reraised exceptions and the unhandled exceptions are subgroups of the
original `ExceptionGroup`, and share its metadata (cause, context, traceback).
On the other hand, each of the explicitly raised exceptions has its own
metadata - the traceback contains the line from which it was raised, its
cause is whatever it may have been explicitly chained to, and its context is the
value of `sys.exc_info()` in the `except*` clause of the raise.

In the aggregated `ExceptionGroup`, the reraised and unhandled exceptions have
the same relative structure as in the original exception, as if they were split
off together in one `subgroup` call. For example, in the snippet below the
inner `try-except*` block raises an `ExceptionGroup` that contains all
`ValueError`s and `TypeError`s merged back into the same shape they had in
the original `ExceptionGroup`:

```python
>>> try:
...   try:
...     raise ExceptionGroup("eg",
...                          [ValueError(1),
...                           TypeError(2),
...                           OSError(3),
...                           ExceptionGroup(
...                              "nested",
...                              [OSError(4), TypeError(5), ValueError(6)])])
...   except *ValueError as e:
...     print(f'*ValueError: {e!r}')
...     raise
...   except *OSError as e:
...     print(f'*OSError: {e!r}')
... except ExceptionGroup as e:
...   print(repr(e))
...
*ValueError: ExceptionGroup('eg', [ValueError(1), ExceptionGroup('nested', [ValueError(6)])])
*OSError: ExceptionGroup('eg', [OSError(3), ExceptionGroup('nested', [OSError(4)])])
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
likely be lost:

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

* It is not possible to use both traditional `except` blocks and the new
`except*` clauses in the same `try` statement. The following example is a
`SyntaxErorr`:

```python
try:
   ...
except ValueError:
   pass
except *CancelledError:  # <- SyntaxError:
   pass                  #    combining `except` and `except*` is prohibited
```

* It is possible to catch the `ExceptionGroup` type with `except`, but not
with `except*` because the latter is ambiguous:

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
error prone and there are clearer ways to implement a check like this:

```python
def foo():
  try:
    raise ExceptionGroup("msg", A(), B())
  except *A:
    return 1   # <- SyntaxError
  except *B as e:
    raise TypeError("Can't have B without A!")
```

## Backwards Compatibility

Backwards compatibility was a requirement of our design, and the changes we
propose in this PEP will not break any existing code:

* The addition of a new builtin exception type `ExceptionGroup` does not impact
existing programs. The way that existing exceptions are handled and displayed
does not change in any way.

* The behaviour of `except` is unchanged so existing code will continue to work.
Programs will only be impacted by the changes proposed in this PEP once they
begin to use `ExceptionGroup`s and `except*`.


Once programs begin to use these features, there will be migration issues to
consider:

* An `except Exception:` clause will not catch `ExceptionGroup`s because they
are derived from `BaseException`. Any such clause will need to be replaced
by `except (Exception, ExceptionGroup):` or `except *Exception:`.

* Similarly, any `except T:` clause that wraps code which is now potentially
raising `ExceptionGroup` needs to become `except *T:`, and its body may need
to be updated.

* Libraries that need to support older Python versions will not be able to use
`except*` or raise `ExceptionGroup`s.


## Security Implications

## How to Teach This

## Reference Implementation

We developed these concepts (and the examples for this PEP) with
[an experimental implementation](https://github.com/iritkatriel/cpython/tree/exceptionGroup-stage5).

It has the builtin `ExceptionGroup` along with the changes to the traceback
formatting code, in addition to the grammar and interpreter changes required
to support `except*`.

Two opcodes were added: one implements the exception type match check via
`ExceptionGroup.split()`, and the other is used at the end of a `try-except`
construct to merge all unhandled, raised and reraised exceptions (if any).
The raised/reraised exceptions are collected in a list on the runtime stack.
For this purpose, the body of each `except*` clause is wrapped in a traditional
`try-except` which captures any exceptions raised. Both raised and reraised
exceptions are collected in one list. When the time comes to merge them into
a result, the raised and reraised exceptions are distinguished by comparing
their metadata fields (context, cause, traceback) with those of the originally
raised exception. As mentioned above, the reraised exceptions have the same
metadata as the original, while raised ones do not.

## Rejected Ideas

### The ExceptionGroup API

We considered making `ExceptionGroup`s iterable, so that `list(eg)` would
produce a flattened list of the leaf exceptions contained in the group.
We decided that this would not be not be a sound API, because the metadata
(cause, context and traceback) of the individual exceptions in a group are
incomplete and this could create problems.  If use cases arise where this
can be helpful, we can document (or even provide in the standard library)
a sound recipe for accessing an individual exception: use the `split()`
method to create an `ExceptionGroup` for a single exception and then
extract the contained exception with the correct metadata.

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
to handle both naked exceptions and `ExceptionGroup`s. Its semantics would
be the same as those of `except*` when catching an `ExceptionGroup`, but
it would not wrap a naked exception to create an `ExceptionGroup`. This
would have been part of a long term plan to replace `except` by `catch`,
but we decided that deprecating `except` in favour of an enhanced keyword
would be too confusing for users at this time, so it is more appropriate
to introduce the `except*` syntax for `ExceptionGroup`s while `except`
continues to be used for simple exceptions.

### Applying an `except*` clause on one exception at a time

We considered making `except*` clauses always execute on a single exception,
possibly executing the same clause multiple times when it matches multiple
exceptions. We decided instead to execute each `except*` clause at most once,
giving it an `ExceptionGroup` that contains all matching exceptions. The reason
for this decision was the observation that when a program needs to know the
particular context of an exception it is handling, it handles it before
grouping it with other exceptions and raising them together.

For example, `KeyError` is an exception that typically relates to a certain
operation. Any recovery code would be local to the place where the error
occurred, and would use the traditional `except`:

```python
try:
  dct[key]
except KeyError:
  # handle the exception
```

It is unlikely that asyncio users would want to do something like this:

```python
try:
  async with asyncio.TaskGroup() as g:
    g.create_task(task1); g.create_task(task2)
except *KeyError:
  # handling KeyError here is meaningless, there's
  # no context to do anything with it but to log it.
```

When a program handles a collection of exceptions that were aggregated into
an exception group, it would not typically attempt to recover from any
particular failed operation, but will rather use the types of the errors to
determine how they should impact the program's control flow or what logging
or cleanup is required. This decision is likely to be the same whether the group
contains a single or multiple instances of something like a `KeyboardInterrupt`
or `asyncio.CancelledError`.  Therefore, it is more convenient to handle all
exceptions matching an `except*` at once.  If it does turn out to be necessary,
the handler can inpect the `ExceptionGroup` and process the individual
exceptions in it.

### Not matching naked exceptions in `except*`

We considered the option of making `except *T` match only `ExceptionGroup`s
that contain `T`s, but not naked `T`s. To see why we thought this would not be a
desirable feature, return to the distinction in the previous paragraph between
operation errors and control flow exceptions. If we don't know whether
we should expect naked exceptions or `ExceptionGroup`s from the body of a
`try` block,  then we're not in the position of handling operation errors.
Rather, we are likely calling some callback and will be handling errors to make
control flow decisions. We are likely to do the same thing whether we catch a
naked exception of type `T` or an `ExceptionGroup` with one or more `T`s.
Therefore, the burden of having to explicitly handle both is not likely to have
semantic benefit.

If it does turn out to be necessary to make the distinction, it is always
possible to nest in the `try-except*` clause an additional `try-except` clause
which intercepts and handles a naked exception before the `except*` clause
has a change to wrap it in an  `ExceptionGroup`. In this case the overhead
of specifying both is not additional burden - we really do need to write a
separate code block to handle each case:

```python
try:
  try:
    ...
  except SomeError:
    # handle the naked exception
except *SomeError:
  # handle the ExceptionGroup
```

### Allow mixing `except:` and `except*:` in the same `try`

This option was rejected because it adds complexity without adding useful
semantics. Presumably the intention would be that an `except T:` block handles
only naked exceptions of type `T`, while `except *T:` handles `T` in
`ExceptionGroup`s. We already discussed above why this is unlikely
to be useful in practice, and if it is needed then the nested `try-except`
block can be used instead to achieve the same result.

### `try*` instead of `except*`

Since either all or none of the clauses of a `try` construct are `except*`,
we considered changing the syntax of the `try` instead of all the `except*`
clauses. We rejected this because it would be less obvious. The fact that we
are handling `ExceptionGroup`s of `T` rather than only naked `T`s should be
specified in the same place where we state `T`.

## See Also

* An analysis of how exception groups will likely be used in asyncio
  programs:
  https://github.com/python/exceptiongroups/issues/3#issuecomment-716203284

* The issue where the `except*` concept was first formalized:
  https://github.com/python/exceptiongroups/issues/4


## References

## Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
