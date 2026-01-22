Feature Name: modifies

Start Date: 2026-01-22

##Summary

This RFC introduces a new contractual aspect ``Modifies`` to express which parts of objects
may be changed by subprogram in a fine-grained manner.

##Motivation

This would be useful to specify function that update an object with many field, but only change
a few.


#Guide-Level explanation


The ``Modifies`` aspect can be specified on subprogram entities. It specifies an aggregate,
in which the choices are Boolean expressions, and the components are list of object
subcomponents. It is intended to specify that for the choice evaluating to True on entry,
the subprogram is only allowed to modify the listed subcomponents. The ``others`` choice
is allowed, and denote the conjunction of the negations of other choices of the aggregates
(a Boolean expression expressing that all other choices evaluates to False).

``ada
type R is record
  First_Field : Boolean;
  Second_Field : Boolean;
  Third_Field : Boolean;
  -- Many other fields
  ...
end record;

procedure Write_Field (X : in out First_Field; Flag : Boolean)
  with Modifies => (Flag   => X.First_Field,
                    others => (X.Second_Field, X.Third_Field));
``

The choices may be omitted, in which case an implicit ``others`` choice is assumed.

While looking similar to the ``Global`` aspect, ``Modifies`` is closer to a ` Contract_Case``.
A Modifies aspect is intended to be equivalent to a post-condition specified with a delta
aggregate. For example, the above ``Modifies`` is equivalent to the following post-condition:

``ada
(if Flag'Old then Identical (X, (X'Old with First_Field = X.First_Field))
 else Identical (X, (X'Old with Second_Field = X.Second_Field, Third_Field = X.Third_Field))
``
where we use ``Identical`` to denote that the objects are indistinguishable. This is a stricter
relation than Ada's equality.

## Reference-Level explanation

Syntax

``bnf
with Modifies => (SUBCOMPONENT_LIST | MODIFIES_CASE {,MODIFIES_CASE})

MODIFIES_CASE ::= CASE_GUARD => (SUBCOMPONENT_LIST)
CASE_GUARD ::= boolean_EXPRESSION
SUBCOMPONENT_LIST ::= SUBCOMPONENT {,SUBCOMPONENT}
SUBCOMPONENT ::= subcomponent_NAME
``

A ``Modifies`` aspect (C1 => (U11,...,), C2 => (U21,...,), ..., Cm => (Um1,...,)) is a short-hand for:
+ An assertion at the beginning of the entity's body (before any declarations)
  that exactly one of the conditions ``C1,...,Cm`` is True.
+ For each i, an additional post-condition with ``Assertion_Level => Static`` expressing that if
  ``Ci'Old`` is True, for all objects that the entity is known to possibly modify
  (e.g from ``in out`` parameters and ``Globals``), subcomponents others than the ones listed
  for ``Ci`` are identical to their ``'Old`` value.
+ If the subprogram may raise exceptions, for each i, an additional ``Exceptional_Cases`` contract
  expressing that if ``Ci'Old`` is True, for all objects that the entity is known to possibly
  modify, restricted to by-reference objects in the case of parameters, subcomponents others
  than the ones listed for ``Ci``are identical to their ``'Old`` value.
+ If the subprogram may terminates the partition (e.g subprogram with annotation ``Program_Exit``),
  for each i, an additional ``Program_Exit`` contract expressing that if ``Ci'Old`` is True, for all
  objects listed in the ``Global`` contract, subcomponents others than the ones listed for ``Ci``
  are identical to their ``'Old`` value.