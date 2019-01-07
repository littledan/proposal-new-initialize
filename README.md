# `class.initialize()`

Early draft proposal to support initializing given objects with fields and private methods.

Status: Not yet presented at TC39; not at a stage

## Background

Creating an instance of a JavaScript subclass is a bottom-up process: The subclass constructor is invoked, which is able to return any object it wants. The "normal" thing for it to do would be to call the superclass constructor, and then following that, do any initialization relevant to the subclass. Here's an abstract example:

```js
class A {
  constructor() { this.x = 1; }
}
class B extends A {
  constructor() { super(); this.y = 2; }
}

const instance = new B();
console.log(instance.x);  // 1
console.log(instance.y);  // 2
console.log(instance.z);  // undefined

```

The "super call" `super()` finds the superclass dynamically, according to the [GetSuperConstructor](https://tc39.github.io/ecma262/#sec-getsuperconstructor) algorithm. This algorithm is a fancy way of saying, `super()` calls `B.__proto__`, whatever that is when `super()` executes, rather than always calling out to the original `A`.

So, if you mutate the prototype, a different super constructor will be called, which won't do the original super constructor's action!

```js
B.__proto__ = class { constructor() { this.z = 3; } };
const brokenInstance = new B();
console.log(brokenInstance.x);  // undefined!
console.log(brokenInstance.y);  // 2
console.log(brokenInstance.z);  // 3!
```

You can prevent this from happening by making the prototype chain immutable, with `Object.preventExtensions` or `Object.freeze`:

```js
Object.freeze(B)
B.__proto__ = class { constructor() { this.z = 3; } };  // TypeError
```

However, built-in subclasses tend to call the original superclass, while not freezing the constructor, whether they are from JavaScript or the Web platform, as [this issue](https://github.com/tc39/proposal-class-fields/issues/179) points out. Fortunately, JavaScript provides a power tool to get around the issue: `Reflect.construct`. With `Reflect.construct`, you can specifically call out to a particular superclass constructor, passing `new.target` (the lowest level subclass) as a parameter. With this technique, we'd redefine `B` as follows:

```js
class B extends A {
  constructor() {
    const instance = Reflect.construct(A, [], new.target);
    instance.y = 2;
    return instance;
  }
}

B.__proto__ = class { constructor() { this.z = 3; } };
const instance = new B();
console.log(instance.x);  // 1!
console.log(instance.y);  // 2
console.log(instance.z);  // undefined!
```

This works great, and you can think of it as analogous to how built-in classes work.

## The problem

Field declarations and private methods are added by the language runtime implicitly, either at the beginning of the constructor (in base classes) or as the last step in `super()` (in subclasses). If we're using `Reflect.construct` to make sure we're always calling out to the original superclass constructor, while keeing the constructor itself non-frozen, how should we allow the subclass to add fields or private methods?

```js
class C {
  #v = 1;
  get v() { return this.#v; }
}
class D {
  #w = 2;
  get w() { return this.#w; }
}
const instance = new D();
console.log(instance.v);  // 1
console.log(instance.w);  // 2

D.__proto__ = class {};
const broken = new D();
console.log(broken.v);  // TypeError!
console.log(broken.w);  // 2
```

How should we prevent the TypeError and make sure that we're calling the `C`'s constructor, while still allowing the `D`'s constructor to add the private field `#w`?

## Proposed solution

The `class.initialize()` syntax can be used to add fields and private methods to any object, when a class wants to avoid calling the super constructor. The class `D` could be written as follows:

```js
class D {
  #w = 2;
  get w() { return this.#w; }
  constructor() {
    const instance = Reflect.construct(C, [], new.target);
    class.initialize(instance);
    return instance;
  }
}

D.__proto__ = class {};
const broken = new D();
console.log(broken.v);  // 1!
console.log(broken.w);  // 2
```

## Detailed syntax and semantics

`class.initialize` is only permitted lexically inside of a constructor of a class, or inside of a direct `eval` in a constructor. It evaluates to a built-in function of length 1, which takes its single argument and calls the [`InitializeInstanceElements`](https://tc39.github.io/proposal-private-methods/#initialize-instance-elements) abstract operation on it. The argument is used as the return value.

Note, only the fields and private methods from the class containing the current constructor will be added; fields and private methods from inherited classes are not added by `class.initialize` (but you can produce an object which has them through `Reflect.construct` if you want).

The syntax uses the meta-property pattern, following the examples of `new.target` and `function.sent`. The syntax here overlaps with the [class access expressions proposal](https://github.com/tc39/proposal-class-access-expressions). The idea here would be to use the `static` keyword for that proposal, or some other metaproperty for this proposal. Most keywords don't have any metaproperties defined for them, so I'm confident we can find a non-clashing solution for both proposals.

## Implementation notes

Some current and in-progress implementations in JS engines of public and private fields and methods internally create a hidden function which does the equivalent of `class.initialize`, but taking the instance under construction as the receiver, not the parameter. The implementation of `class.initialize` would be a simple wrapper around that function. It only needs to be created if the constructor contains `class.initialize` or a direct `eval` lexically contained in it, so there should not be significant memory overhead in practice.

## Interim solutions

If you want to get the benefit of this proposal without waiting for it to move through standardization, you can use one of the following strategies for now:
- Define private fields or methods in the base class (arguably this is what the built-in classes are doing), and replace private fields in in subclasses with writes to properties (e.g., by transpiling them away)
- Use WeakMap and WeakSet instead of private fields and methods, so that you can put them in place with the `Reflect.construct` pattern. A transpiler implementation of fields and private methods and this proposal will likely do the same.
- Prevent prototype mutation by `Object.preventExtensions` or `Object.freeze` on the subclass constructor.

Because these strategies seem sufficient to start, even as they each have their limitations, this proposal does not need to block the class fields proposal.

## Credits

Thanks to Allen Wirfs-Brock for raising this issue some years ago, and Domenic Denicola for providing the context of application for polyfilled standard library features.
