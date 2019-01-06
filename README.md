# `class.initialize()`

Early draft proposal to support initializing given objects with fields and private methods.

Status: Not yet presented at TC39; not at a stage

## The problem

ES2015 subclassing works by 

https://github.com/tc39/proposal-class-fields/issues/179

## Proposed solution

The `class.initialize()` syntax can be used to add fields and private methods to any object, when a class wants to avoid calling the super constructor.

## Detailed syntax and semantics

`class.initialize` is only permitted lexically inside of a constructor of a class, or inside of a direct `eval` in a constructor. It evaluates to a built-in function of length 1, which takes its single argument and calls the [`InitializeInstanceElements`](https://tc39.github.io/proposal-private-methods/#initialize-instance-elements) abstract operation on it. The argument is used as the return value.

The syntax uses the meta-property pattern, following the examples of `new.target` and `function.sent`. The syntax here overlaps with the [class access expressions proposal](https://github.com/tc39/proposal-class-access-expressions). The idea here would be to use the `static` keyword for that proposal, or some other metaproperty for this proposal. Most keywords don't have any metaproperties defined for them, so I'm confident we can find a non-clashing solution for both proposals.

## Implementation notes

Some current and in-progress implementations in JS engines of public and private fields and methods internally create a hidden function which does the equivalent of `class.initialize`, but taking the instance under construction as the receiver, not the parameter. The implementation of `class.initialize` would be a simple wrapper around that function. It only needs to be created if the constructor contains `class.initialize` or a direct `eval` lexically contained in it, so there should not be significant memory overhead in practice.

## Staging

This document is a separate proposal from the class fields proposal. It will be useful in conjunction with it, but is not necessary to block the other proposal. As an intermediate strategy, code which wants to be defensive against constructor prototype mutation can either use `Object.preventExtensions` or `Object.freeze` on the subclass constructor to block this mutation, or it can define its private fields and methods in the base class.

## Credits

Thanks to Allen Wirfs-Brock for raising this issue some years ago, and Domenic Denicola for providing the context of application for polyfilled standard library features.
