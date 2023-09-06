PEP: <REQUIRED: pep number>
Title: <REQUIRED: pep title>
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

PEP 484 defines the ``@typing.overload`` decorator, which allows a type checker to understand all possible signatures for a function.
Since ``typing`` decorators are no-ops at runtime, the actual function's logic must stays within the last, non-decorated function declaration
to be compliant with type checkers.
This has the unintended consequence of making it difficult for multiple-dispatch libraries to integrate seamlessly with static type checkers.

Motivation
==========

While multiple dispatch and overloading are two different concept, it could be possible to bring them together.

Consider an example from the Mypy documentation::

  from typing import Union, overload

  @overload
  def mouse_event(x1: int, y1: int) -> ClickEvent: ...
  @overload
  def mouse_event(x1: int, y1: int, x2: int, y2: int) -> DragEvent: ...

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

Interest in multiple dispatch has been there since the early days of Python. Over the years, many implementation appeared, proving 
that the community liked and wish for this pattern to be possible:

* https://peps.python.org/pep-0443/
* https://github.com/mrocklin/multipledispatch/
* https://github.com/coady/multimethod
* https://github.com/beartype/plum
* https://github.com/erezsh/runtype
* https://github.com/gabrieldemarmiesse/overtake

Rationale
=========

[Describe why particular design decisions were made.]


Specification
=============

This PEP proposes the introduction of a new ``multiple_dispatcher`` decorator within the ``typing`` module.
While it will have no runtime effect, type-checkers will recognize it as a marker indicating the application of a multiple
dispatch mechanism. Any function decorated with ``@typing.multiple_dispatcher`` will be 
expected decorate another function that adheres to the
PEP 484 conventions concerning ``@typing.overload``.

An illustrative example is provided below::

  from typing import overload, get_overloads, multiple_dispatcher, TypeVar, TypeSpec

  T = TypeVar("T")
  P = ParamSpec("P")

  @multiple_dispatcher
  def some_multiple_dispatch_mechanism(func: Callable[P, T]) -> Callable[P, T]:
      """This function will very likely be declared in a Python package"""
      def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
          for overloaded_func in get_overload(func):
              if args_kwargs_are_matching_the_overload(args, kwargs, overloaded_func):
                  return overloaded_func(*args, **kwargs)
          raise TypeError("Bad arguments")
      return wrapper


  @overload
  def mouse_event(x1: int, y1: int) -> ClickEvent:
      return ClickEvent(x1, y1)

  @overload
  def mouse_event(x1: int, y1: int, x2: int, y2: int) -> DragEvent:
      return DragEvent(x1, y1, x2, y2)

  @some_multiple_dispatch_mechanism
  def mouse_event(x1, y1, x2=None, y2=None):
      ...

Upon detecting the ``some_multiple_dispatch_mechanism`` decorator, type-checkers should infer that functions decorated with ``@overload`` will be executed following the PEP 484 rules for signature resolution. Consequently:

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

By having type-checkers and IDEs understanding user's code, we can warn users if they are not filling the right functions.

The multiple dispatch behavior must be taught by third-party libraries. The ``multiple_dispatcher`` decorator must be
documented in the standard library and is mostly aimed at library authors, who are rarely novices.


Reference Implementation
========================

Overtake: A library that makes multiple dispatch work with ``@overload``: https://github.com/gabrieldemarmiesse/overtake

Mypy: Currently works well with Overtake without any special decorator: https://github.com/python/mypy
Nonetheless, Mypy may in the future decide to enforce the rule about ``@overload`` functions being empty (this is why this PEP
exists).
Additionally, should this PEP be accepted, Mypy could enforce the functions having an empty body depending on the presence
or absence of a multiple dispatch library.


Rejected Ideas
==============

Implement multiple dispatch in the standard library: Too much work, we can always make another PEP about it later.

Choose the status quo: While Mypy works with the reference implementation of a multiple dispatch library, that's only because it
does not enforce all the rules about the body of functions decorated by ``@overload`` described in PEP 484.

Loosen the requirements about the body of overloaded functions being empty.
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

We could rename the decorator. ``multiple_dispatcher`` is good but the name can be better I believe.


Footnotes
=========

Many thanks for Michael Chow, Wessel Bruinsma and Nicolas Tessore for providing awesome ideas!


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
