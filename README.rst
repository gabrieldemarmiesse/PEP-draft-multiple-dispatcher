PEP: <REQUIRED: pep number>
Title: Support multiple dispatch libraries in Python's type system
Author: Gabriel de Marmiesse <gabrieldemarmiesse@gmail.com>
Sponsor: <real name of sponsor>
PEP-Delegate: <PEP delegate's real name>
Discussions-To: <REQUIRED: URL of current canonical discussion thread>
Status: Draft
Type: Standards Track
Topic: Typing
Content-Type: text/x-rst
Requires: <pep numbers>
Created: 06-Sep-20123
Python-Version: 3.13
Post-History: <REQUIRED: dates, in dd-mmm-yyyy format, and corresponding links to PEP discussion threads>
Resolution: <url>


Abstract
========

:pep:`484` defines the ``@typing.overload`` decorator, which allows a type checker to understand all possible signatures for a function.
Since ``typing`` decorators are no-ops at runtime, the actual function's logic must stays within the last, non-decorated function declaration
to be compliant with type checkers.
This has the unintended consequence of making it difficult for multiple-dispatch libraries leveraging ``@typing.overload`` to integrate seamlessly with static type checkers.
This PEP proposes a new decorator, being no-op at runtime: ``@typing.multiple_dispatcher`` that informs type-checkers that we are working
with a multiple dispatch library. The library must respect the signature resolution rules defined in :pep:`484`.

Motivation
==========

While multiple dispatch and overloading are two different concept, it could be possible to bring them together.

Consider this example from the Mypy documentation::

  from typing import Union, overload

  @overload
  def mouse_event(x1: int, y1: int) -> ClickEvent:
      ...

  @overload
  def mouse_event(x1: int, y1: int, x2: int, y2: int) -> DragEvent:
      ...

  def mouse_event(x1: int,
                  y1: int,
                  x2: Optional[int] = None,
                  y2: Optional[int] = None) -> Union[ClickEvent, DragEvent]:
      if x2 is None and y2 is None:
          return ClickEvent(x1, y1)
      elif x2 is not None and y2 is not None:
          return DragEvent(x1, y1, x2, y2)
      else:
          raise TypeError("Bad arguments")

Such syntax breaches the DRY (Don't Repeat Yourself) principle, as it can be challenging to ensure
the declared signatures consistently match the internal if-else logic.
A novice or someone transitioning from languages like C++ or Julia might find the following syntax more intuitive::

  def mouse_event(x1: int, y1: int) -> ClickEvent:
      return ClickEvent(x1, y1)

  def mouse_event(x1: int, y1: int, x2: int, y2: int) -> DragEvent:
      return DragEvent(x1, y1, x2, y2)

While it might be to complicated for Python or the standard library to find the right function to call depending
on the argument types at runtime, it is possible for third-party libraries to tackle the issue (see reference implementation section).

Interest in multiple dispatch has been there since the early days of Python. Over the years, many implementations appeared, proving
that the community liked and wished for this pattern to be possible:

* https://peps.python.org/pep-0443/
* https://github.com/mrocklin/multipledispatch/
* https://github.com/coady/multimethod
* https://github.com/beartype/plum
* https://github.com/erezsh/runtype
* https://github.com/gaphor/generic
* https://gnosis.cx/publish/programming/charming_python_b12.html
* https://github.com/gabrieldemarmiesse/overtake

One issue is there though, while libraries can implement multiple dispatch, type-checkers are using the :pep:`484` rules about overload.
One of those rules, that is causing us issues here is the following::

  > The @overload-decorated definitions are for the benefit of the type checker only,
  > since they will be overwritten by the non-@overload-decorated definition, while
  > the latter is used at runtime but should be ignored by a type checker.
  > At runtime, calling a @overload-decorated function directly will raise NotImplementedError.

While ``typing.get_overloads()`` freed us from those constraints at runtime, static type-checkers still work with this assumption.

Rationale
=========

We can expose in this section a list of requirements that we want to satisfy with our proposal.

#. Create a mechanism that will allow type-checkers to know that a multiple-dispatch library is used.
#. This mechanism should be invisible and automatic to multiple-dispatch libraries' users.
#. The change should be backward compatible.
#. The change should have no run-time effect.
#. This mechanism shouldn't be configurable, because the ``@typing.overload`` is not configurable.

The next section "Specification" gives a new change in the language that respects those requirements.

Specification
=============

This PEP proposes the introduction of a new ``multiple_dispatcher`` decorator within the ``typing`` module.
While it will have no runtime effect, type-checkers will recognize it as a marker indicating the application of a multiple
dispatch mechanism. Any function decorated with ``@typing.multiple_dispatcher`` will be
expected decorate another function that adheres to the
:pep:`484` conventions concerning ``@typing.overload``.

An illustrative example is provided below::

  from typing import get_overloads, TypeVar, TypeSpec, Callable

  T = TypeVar("T")
  P = ParamSpec("P")

  @multiple_dispatcher
  def some_multiple_dispatch_mechanism(func: Callable[P, T]) -> Callable[P, T]:
      """This function will very likely be declared in a Python package"""
      def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
          """The actual implementation is up to the library's author, here is a dummy one."""
          for overloaded_func in get_overloads(func):
              if args_kwargs_are_matching_the_overload(args, kwargs, overloaded_func):
                  return overloaded_func(*args, **kwargs)
          raise TypeError("Bad arguments")
      return wrapper

The decorator can then be used by end users, and will be understood by type-checkers::

  from typing import overload

  from my_module import some_multiple_dispatch_mechanism

  @overload
  def mouse_event(x1: int, y1: int) -> ClickEvent:
      return ClickEvent(x1, y1)  # type checkers are ok with the body being filled

  @overload
  def mouse_event(x1: int, y1: int, x2: int, y2: int) -> DragEvent:
      return DragEvent(x1, y1, x2, y2)  # type checkers are ok with the body being filled

  @some_multiple_dispatch_mechanism
  def mouse_event(x1, y1, x2=None, y2=None):
      raise NotImplementedError  # type checkers are ok with the body being empty

Upon detecting the ``some_multiple_dispatch_mechanism`` decorator, type-checkers should understand that functions decorated with ``@overload`` will be executed following the :pep:`484` rules for signature resolution.
Consequently:

The body of functions decorated with ``@overload`` should not be empty.
The final function's body, typically containing the logic in the traditional overload mechanism before this PEP, should be empty.
Using ``...`` or a simple ``raise NotImplementedError`` would suffice.

Backwards Compatibility
=======================

This PEP is backward compatible and has no influence on any existing working code, since the behavior of Python and the type-checkers does not change without the ``@multiple_dispatcher`` decorator.

Security Implications
=====================

This might not be relevant.

How to Teach This
=================

We will not focus here on teaching library authors to use ``@typing.multiple_dispatcher`` as it's quite trivial
and they are very few. The documentation about ``typing.get_overloads`` will include a mention and a
link to ``@typing.multiple_dispatcher`` since it's typically what library authors will use. This should be enough.

The next part will focus on end-users.

This PEP will encourage multiple dispatch libraries to leverage ``typing.overload`` and ``typing.get_overloads``.
As such, we will end up with users having to fill the functions decorated with ``@overload`` (doing multiple dispatch) and
users filling only the last function (without multiple dispatch).

Having the two pattern on stackoverflow, github, etc... may confuse newcomers and we should address this as we do not wish
for them to fill the wrong functions and get silent errors (empty function bodies being called by mistake).

We propose to change the documentation examples and encourage users to use ``raise NotImplementedError`` instead of ``...`` to
implement the empty function body. This should deal with silent errors. Users may still use ``...`` if they feel confident
about the correctness of their code, this is still considered as an "empty body" by type-checkers.

Ideally, IDEs and type checkers should help with this issue too here.
By having type-checkers and IDEs understanding user's code, they can understand if a multiple dispatch library is used, then
they can warn users if they are not filling the right functions.


Reference Implementation
========================

Overtake: A library that makes multiple dispatch work with ``@overload``: https://github.com/gabrieldemarmiesse/overtake

Mypy: Currently works well with Overtake without any special decorator: https://github.com/python/mypy
Nonetheless, Mypy may in the future decide to enforce the rule about ``@overload`` functions being empty. This rule is currently enforced
by Pyright. this is why this PEP exists.
Additionally, should this PEP be accepted, Mypy could enforce the functions having an empty body depending on the presence
or absence of a multiple dispatch library.


Rejected Ideas
==============

Implement multiple dispatch in the standard library:
----------------------------------------------------

Too much work, we can always make another PEP about it later.

Choose the status quo:
----------------------

While Mypy works with the reference implementation of a multiple dispatch library, that's only because it
does not enforce all the rules about the body of functions decorated by ``@overload`` described in PEP 484.

Loosen the requirements about the body of overloaded functions being empty:
---------------------------------------------------------------------------

While we could remove this requirement in
the type checkers and call it a day, the type checker cannot then warn the user that the code is not at the right place.
This is an easily preventable error by type checkers.
The type checker has then to special case this type of function to avoid triggering the error about return value not being present
since it can't know if we are using a multiple dispatch library.
Consider this example::

  from typing import Union, overload

  @overload
  def mouse_event(x1: int, y1: int) -> ClickEvent:
      ...  # how does this not raise an error "ClickEvent is not returned"?

  @overload
  def mouse_event(x1: int, y1: int, x2: int, y2: int) -> DragEvent:
        ...  # how does this not raise an error "DragEvent is not returned"?

  def mouse_event(x1: int,
                  y1: int,
                  x2: Optional[int] = None,
                  y2: Optional[int] = None) -> Union[ClickEvent, DragEvent]:
      # Here with a multiple dispatch library, the body would be empty, so the type checker,
      # to avoid throwing an error with "Union[ClickEvent, DragEvent] is not returned"
      # would have to implement additional logic.

      if x2 is None and y2 is None:
          return ClickEvent(x1, y1)
      elif x2 is not None and y2 is not None:
          return DragEvent(x1, y1, x2, y2)
      else:
          raise TypeError("Bad arguments")


Open Issues
===========

We could rename the decorator. ``multiple_dispatcher`` is good but the name can be better.


Footnotes
=========

Many thanks for Michael Chow, Wessel Bruinsma and Nicolas Tessore for providing awesome ideas!


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
