# Introducing try..except* syntax


## Abstract

This PEP proposes language extensions that allow programs to raise and handle
multiple unrelated exceptions simultaneously:

* A new standard exception type, the `ExceptionGroup`, which represents a
group of unrelated exceptions being propagated together.

* A new syntax `except*` for handling `ExceptionGroup`s.

## Motivation

The interpreter is currently able to propagate at most one exception at a
time. The chaining features introduced in PEP3134 [reference] link together
exceptions that are related to each other as the cause or context, but there
are situations where there are multiple unrelated exceptions that need to be
propagated together as the stack unwinds. Several real world use cases are
listed below.

[TODO: flesh these out]
* asyncio programs, trio, etc

* Multiple errors from separate retries of an operation [https://bugs.python.org/issue29980]

* Situations where multiple unrelated exceptions may be of interest to calling code [https://bugs.python.org/issue40857]

* Multiple teardowns in pytest raising exceptions [https://github.com/pytest-dev/pytest/issues/8217]

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
case of `ExceptionGroup("three")` in the example above.

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

The original `eg` is unchanged by `subgroup` or `split`. If it, or any
nested `ExceptionGroup` is not included in the result in full, a new
`ExceptionGroup` is created, containing a subset of the exceptions in
its `errors` list. This partition is done recursively, so potentially
the entire exception tree is copied. There is no need to copy the leaf
exceptions and the metadata elements (cause, context, traceback).

If a split is trivial (one side is empty), then None is returned for the
other size:

```python
>>> other_errors.split(lambda e: isinstance(e, SyntaxError))
(None, ExceptionGroup('one', [ExceptionGroup('two', [ValueError(3)]), ExceptionGroup('three', [OSError(4)])]))
```

Since splitting by exception type is a very common use case, `subgroup` and
`split` can take an exception type or tuple of exception types and treat it
as a shorthand for matching that type: `eg.split(TypeError)`, is equivalent to
`eg.split(lambda e: isinstance(e, TypeError))`.


#### The Traceback of and `ExceptionGroup`

For regular exceptions, the traceback represents a simple path of frames,
from the frame in which the exception was raised to the frame in which it was
was caught or, if it hasn't been caught yet, the frame that the program's
execution is currently in. The list is constructed by the interpreter, which
appends any frame from which it exits to the traceback of the 'current
exception' if one exists (the exception returned by `sys.exc_info()`). To
support efficient appends, the links in a traceback's list of frames are from
the oldest to the newest frame. Appending a new frame is then simply a matter
of inserting a new head to the linked list referenced from the exception's
`__traceback__` field. Crucially, the traceback's frame list is immutable in
the sense that frames only need to be added at the head, and never need to be
removed.

We do not need to make any changes to this data structure. The `__traceback__`
field of the ExceptionGroup object represents the path that the exceptions
travelled through together after being joined into the `ExceptionGroup`, and
the same field on each of the nested exceptions represents that path through
which each exception arrived to the frame of the merge.

What we do need to change is any code that interprets and displays tracebacks,
because it will now need to continue into tracebacks of nested exceptions
once the traceback of an ExceptionGroup has been processed. For example:


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

 In the following we use the term "naked" exception for regular Python
 exceptions **not wrapped** in an `ExceptionGroup`. E.g. a regular
 `ValueError` propagating through the stack is "naked".

The `except *SpamError` block will be run if the `try` code raised an
`ExceptionGroup` with one or more instances of `SpamError`. It would also be
triggered if a naked instance of  `SpamError` was raised.

The `except *BazError as e` block would create an ExceptionGroup with the
same nested structure and metadata (msg, cause, context and traceback) as the
one raised, but containing only the instances of `BazError`. This
`ExceptionGroup` is assigned to `e`. The type of `e` would be
`ExceptionGroup[BazError]`. If there was just one naked instance of `BazError`,
it would be wrapped into an `ExceptionGroup` and assigned to `e`.

The `except *(BarError, FooError) as e` would split out all instances of
`BarError` or `FooError`  into such an ExceptionGroup and assign it to `e`.
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
except *CancelledError:  # <- SyntaxError:
   pass                  #    combining `except` and `except*` is prohibited
```

It is possible to catch the `ExceptionGroup` type with a plain except, but not
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

### Unmatched Exceptions

Example:

```python
try:
  raise ExceptionGroup(
    "msg", [ValueError('a'), TypeError('b'), TypeError('c'), KeyError('e')]
  )
except *ValueError as e:
  print(f'got some ValueErrors: {e!r}')
except *TypeError as e:
  print(f'got some TypeErrors: {e!r}')
  raise
```

The above code would print:

```
got some ValueErrors: ExceptionGroup('msg', [ValueError('a')])
got some TypeErrors: ExceptionGroup('msg', [TypeError('b'), TypeError('c')])
```

and then terminate raising the following `ExceptionGroup` containing the
exceptions that were not handled or were caught and reraised:

```
ExceptionGroup(
  "msg",
  [KeyError('e'),     # unhandled
   TypeError('b'),    # reraised
   TypeError('c')]    # reraised
)
```

Basically, before interpreting `except *` clauses, the interpreter will
have an "incoming" `ExceptionGroup` object with a list of exceptions in it
to handle, and then:

* The interpreter creates two new empty result lists for any exceptions that
will be raised in the except* blocks: a "reraised" list for the naked raises
and a "raised" list of the parameterised raises. It also saves the original
"incoming" group which it uses in the end, if necessary, to combine the
reraised exceptions into the same structure.

* Every `except *` clause, run from top to bottom, can match a subset of the
  exceptions out of the "incoming" group, forming a "working set" of errors for
  the current clause (the value of `sys.exc_info()` while the clause is
  executed).  These exceptions are removed from the "incoming" group so
  that they will be excluded from matching additional clauses.
  If an except block raises an exception, that exception is added to the
  appropriate result list ("raised" or "reraised"), and in the case of "raise"
  it gets its "working set" of errors linked to it via the `__context__`
  attribute.

* After all `except*` clause were evaluated, there are the following
 possibilities:

1. Both the "incoming" `ExceptionGroup` and the two result lists are empty.
This means that all exceptions were processed and silenced.

2. The "incoming" and/or "reraised" `ExceptionGroup`s are non-empty: not all
exceptions were processed, and some were reraised. The interpreter constructs
a combined subgroup of the original `ExceptionGroup`, consisting of the
"incoming" and the "reraised" exceptions (using the `subgroup` operation to
  preserve the structure of the original `ExceptionGroup`).

3. The list of "raised" exceptions is not empty. Then the interpreter
constructs a new `ExceptionGroup` with an empty message and an exception list
that contains all exceptions in "raised", as well as the `ExceptionGroup`
created in the previous step for any "incoming" or "reraised" exceptions.


The following example shows that order of `except*` clauses is significant
just like with the regular `try..except`, and that a naked exception is
wrapped by an `ExceptionGroup` when caught by an `except *`:

```python
>>> try:
...    raise BlockingIOError
... except *OSError as e:   # Would catch the error
...   print(repr(e))
... except *BlockingIOError: # Would never run
...   print('never')
...
ExceptionGroup('', [BlockingIOError()])
```

### Raising ExceptionGroups explicitly

Exception groups can be derived from other exception groups and raised as follows:

```python
try:
  low_level_os_operation()
except *OSerror as errors:
  raise errors.subgroup(lambda e: e.errno != errno.EPIPE)
```

The above code ignores all `EPIPE` OS errors, while letting all other
exceptions propagate.

Raising exceptions while handling an `ExceptionGroup` introduces nesting
because the traceback and chaining information need to be maintained:

```python
>>> try:
...   try:
...     raise ExceptionGroup("one", [ValueError('a'), TypeError('b')])
...   except *ValueError:
...     raise ExceptionGroup("two", [KeyError('x'), KeyError('y')])
... except BaseException as e:
...   traceback.print_exception(e)
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
>>>
```

A regular `raise Exception` would not wrap `Exception` in its own group, but a
new group would still be created to merged it with the ExceptionGroup of
unhandled exceptions:

```python
>>> try:
...   try:
...     raise ExceptionGroup("eg", [ValueError('a'), TypeError('b')])
...   except *ValueError:
...     raise KeyError('x')
... except BaseException as e:
...   traceback.print_exception(e)
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

### Exception Chaining

If an error occurs during processing a set of exceptions in a `except *`
block, all matched errors would be put in a new `ExceptionGroup` which would
be referenced from the just occurred exception via its `__context__`
attribute:

```python
>>> try:
...   try:
...     raise ExceptionGroup("eg", [ValueError('a'), ValueError('b'), TypeError('z')])
...   except *ValueError:
...     1/0
... except BaseException as e:
...   traceback.print_exception(e)
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
      ------------------------------------------------------------
      ValueError: b

   During handling of the above exception, another exception occurred:

   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
   ZeroDivisionError: division by zero
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: eg
      ------------------------------------------------------------
      TypeError: z
>>>
```

It's also possible to explicitly chain the cause of an exception:

```python
>>> try:
...   try:
...     raise ExceptionGroup("eg", [ValueError('a'), ValueError('b'), TypeError('z')])
...   except *ValueError as errors:
...     raise RuntimeError('unexpected values') from errors
... except ExceptionGroup as e:
...   traceback.print_exception(e)
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
      ------------------------------------------------------------
      ValueError: b

   The above exception was the direct cause of the following exception:

   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
   RuntimeError: unexpected values
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
   ExceptionGroup: eg
      ------------------------------------------------------------
      TypeError: z
>>>
```

### Recursive Matching

The matching of `except *` clauses against an `ExceptionGroup` is performed
recursively, using the `ExceptionGroup.split()` method. E.g.:

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


### Re-raising ExceptionGroups

It is important to point out that the `ExceptionGroup` bound to `e` is an
ephemeral object. Raising it via `raise` or `raise e` will not cause changes
to the overall shape of the `ExceptionGroup`.  Any modifications to it will
likely get lost:

```python
try:
  raise ExceptionGroup(
    "top",
    [ValueError('a'),
     TypeError('b'),
     ExceptionGroup("nested",[TypeError('c'), KeyError('d')])
    ]
  )
except *TypeError as e:
  e.foo = 'bar'
  # ^----------- `e` is an ephemeral object that might get
  #              destroyed after the `except*` clause.
```


With the regular exceptions, there's a subtle difference between `raise e`
and a bare `raise`:

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

* The `raise e` form re-raises the `ExceptionGroup` `e` with its traceback
  updated to point out to the current frame, effectively resulting in user
  seeing the `raise e` line in their tracebacks.

After all `except *` blocks have been processed, the remaning unhandled
exceptions are merged together with the raised and re-reaised exceptions,
and the manner in which this is done depends on what the traceback needs
to contain: in the case of `raise e`, we have a new `ExceptionGroup` that
is merged with the unhandled `ExceptionGroup`, whereas in the case of
a naked `raise` we retain the re-reaised exceptions as if they were
unhandled:

```python

eg = ExceptionGroup(
    "one",
    [ValueError('a'),
     TypeError('b'),
     ExceptionGroup("two", [TypeError('c'), KeyError('d')])
    ]
  )

>>> try:
...   try:
...     raise eg
...   except *TypeError as e:
...     raise
... except ExceptionGroup as e:
...   traceback.print_exception(e)
...
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ExceptionGroup: one
   ------------------------------------------------------------
   ValueError: a
   ------------------------------------------------------------
   TypeError: b
   ------------------------------------------------------------
   ExceptionGroup: two
      ------------------------------------------------------------
      TypeError: c
      ------------------------------------------------------------
      KeyError: 'd'
>>>
>>> try:
...   try:
...     raise eg
...   except *TypeError as e:
...     raise e
... except ExceptionGroup as e:
...   traceback.print_exception(e)
...
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
  File "<stdin>", line 3, in <module>
ExceptionGroup
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 5, in <module>
     File "<stdin>", line 3, in <module>
     File "<stdin>", line 3, in <module>
   ExceptionGroup: one
      ------------------------------------------------------------
      TypeError: b
      ------------------------------------------------------------
      ExceptionGroup: two
         ------------------------------------------------------------
         TypeError: c
   ------------------------------------------------------------
   Traceback (most recent call last):
     File "<stdin>", line 3, in <module>
     File "<stdin>", line 3, in <module>
   ExceptionGroup: one
      ------------------------------------------------------------
      ValueError: a
      ------------------------------------------------------------
      ExceptionGroup: two
         ------------------------------------------------------------
         KeyError: 'd'
>>>
```


### "continue", "break", and "return" in "except*"

`continue`, `break`, and `return` are disallowed in `except*` clauses,
causing a `SyntaxError`.

Consider if they were allowed:

```python
def foo():
  try:
    raise ExceptionGroup("msg", A(), B())
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

## Backwards Compatibility

The behaviour of `except` is unchanged so existing code will continue to work.

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


## Security Implications

## How to Teach This

## Reference Implementation

[An experimental implementation](https://github.com/iritkatriel/cpython/tree/exceptionGroup-stage4).

(`raise` in `except*` not supported yet).

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
