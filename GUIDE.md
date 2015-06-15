# Setup

Node

```sh
npm install tcomb
```

# The idea

What's a type? In tcomb **a type is represented by a function** `T` such that:

1. has signature `T(value)` where value depends on the nature of `T`
2. is idempotent, that is `T(T(value)) = T(value)`
3. owns a static function `T.is(x)` returning `true` if `x` is an instance of `T`

# Irreducibles types

An *irreducible* type is a type that can't be built with other types. Examples of such types are JavaScript native types:

**JavaScript native types**

* `Str`: strings
* `Num`: numbers
* `Bool`: booleans
* `Arr`: arrays
* `Obj`: plain objects
* `Func`: functions
* `Err`: errors
* `Re`: regular expressions
* `Dat`: dates

There are 2 additional irriducible types defined in tcomb:

**Additional types**

* `Nil`: `null` or `undefined`
* `Any`: any type

## Type checking with the `is` function

Every type defined with tcomb owns a static predicate `is(x: any) -> boolean` useful for type checking:

```js
const t = require('tcomb');

t.Str.is('a string'); // => true
t.Str.is(1);          // => false

t.Num.is('a string'); // => false
t.Num.is(1);          // => true

// and so on...
```

## Asserts

tcomb provides a built-in `t.assert(guard: boolean, message?: string)` function, if an assert fails a `TypeError` is thrown.

```js
assert(Str.is('a string')); // => ok
assert(Str.is(1)); // => throws
```

* `guard` is a boolean condition
* `message` optional string useful for debugging

```js
const x = -2;
const min = 0;
// throws "-2 should be greater then 0"
assert(x > min, `${x} should be greater then ${min}`);
```

Another way to ensure the correct type is to use types as constructors:

```js
const s1 = Str('a string'); // => ok
const s2 = Str(1); // => throws
```

## Adding safety to legacy code

tcomb can also be used in existing code to add type safety:

```js
// plain old JavaScript class
function Point (x, y) {
  this.x = x;
  this.y = y;
}

const p = new Point(1, 'a'); // silent error

// Now with asserts inserted:

function Point (x, y) {
  this.x = Num(x);
  this.y = Num(y);
}

const p = new Point(1, 'a'); // => throws
```

## Defining new irreducibles

To define your own irreducible types use the `t.irreducible(name: string, predicate: (x: any) => boolean)` combinator:

```js
const t = require('tcomb');
const React = require('react');

const ReactElement = t.irreducible('ReactElement', React.isValidElement);

ReactElement.is(<div/>); // => true
```

## The meta object

Every type owns a `meta` object containing the following properties:

* `kind`: the type kind, equal to `"irreducible"` for irreducible types
* `is`: the predicate

Example: the `meta` object of `Str`:

```js
{
  kind: 'irreducible',
  is: function isStr(x) {
    return typeof x === 'string';
  }
}
```

**Tech note**. All the built-in irreducible types are defined with `t.irreducible()`.

> Meta objects are a distinctive feature of tcomb, allowing **runtime type introspection**.

# Type combinators

*Type combinators* are the tcomb way to define new composite types from those already defined, that is they **combine** old types in a new one.

## The subtype combinator

You can refine a type using the `subtype(type, predicate, name)` combinator where:

* `type` is a type already defined
* `predicate` is a predicate
* `name` is an optional string useful for debugging purposes

Example:

```js
// defines a type representing positive numbers
const Positive = t.subtype(t.Num, n => return n >= 0, 'Positive');

Positive.is(1);  // => true
Positive.is(-1); // => false
```

Subtypes have the following `meta` object:

```js
{
  kind: 'subtype',
  type: type,
  predicate: predicate
}
```

## The enums combinator

You can define an enum type using the `enums(map: Object, name?: string)` combinator where:

* `map` is a hash whose keys are the enums (values are free)
* `name` is an optional string useful for debugging purposes

```js
const Country = t.enums({
  IT: 'Italy',
  US: 'United States'
}, 'Country');

Country.is('IT'); // => true
Country.is('FR'); // => false
```

Enums have the following `meta` object:

```js
{
  kind: 'enums',
  map: map
}
```

If you don't care of values you can use `enums.of(keys, name?)` where:

*   `keys: Array<Str|Number> | Str` is the array of enums or a string where the enums are separated by spaces
*   `name: ?Str` is an optional string useful for debugging purposes

```js
// values will mirror the keys
const Country = t.enums.of('IT US', 'Country');

// same as
const Country = t.enums(['IT', 'US'], 'Country');

// same as
const Country = t.enums({
  IT: 'IT',
  US: 'US'
}, 'Country');
```

## The struct combinator

You can define a struct type using the `struct(props, name?)` combinator where:

* `props` is a hash whose keys are the field names and the values are the fields types
* `name` is an optional string useful for debugging purposes

```js
const Point = t.struct({
  x: Num,
  y: Num
}, 'Point');

// constructor usage, `p` is immutable, new is optional
const p = new Point({x: 1, y: 2});

Point.is(p); // => true
```

**Tech note**. `Point.is` uses `instanceof` internally.

### Methods

Struct methods are defined as usual:

```js
Point.prototype.toString = function () {
  return `(${this.x}, ${this.y})`;
};
```

Structs have the following `meta` object:

```js
{
  kind: 'struct',
  props: props
}
```

### Extending a struct

Every struct constructor owns an `extend(mixins, name)` function where:

*   `mixins` can be an object containing the new props, an array of objects, a type or an array of types
*   `name` the name of the new struct

```js
const Point3D = Point.extend({z: Num}, 'Point3D');

// multiple inheritance
const A = struct({...});
const B = struct({...});
const MixinC = {...};
const MixinD = {...};
const E = A.extend([B, MixinC, MixinD]);
```

`extend` supports **prototypal inheritance**:

```js
const Rectangle = struct({
  width: Num,
  height: Num
});

Rectangle.prototype.getArea = function () {
  return this.width * this.height;
};

const Cube = Rectangle.extend({
  thickness: Num
});

// typeof Cube.prototype.getArea === 'function'
Cube.prototype.getVolume = function () {
  return this.getArea() * this.thickness;
};
```

> **Note**. Repeated props are not allowed:

```js
const Wrong = Point.extend({x: Num}); // => throws
```

## The tuple combinator

You can define a tuple type using the `tuple(types, name)` combinator where:

* `types` is a list of types
* `name` is an optional string useful for debugging purposes

Instances of tuples are plain old JavaScript arrays.

```js
const Area = tuple([Num, Num]);

// constructor usage, `area` is immutable
const area = Area([1, 2]);
```

Tuples have the following `meta` object:

```js
{
  kind: 'tuple',
  types: types
}
```

## The list combinator

You can define a list type using the `list(type, name)` combinator where:

* `type` is the type of list items
* `name` is an optional string useful for debugging purposes

Instances of lists are plain old JavaScript arrays.

```js
const Path = list(Point);

// costructor usage, `path` is immutable
const path = Path([
  {x: 0, y: 0}, // tcomb hydrates automatically using the `Point` constructor
  {x: 1, y: 1}
]);
```

Lists have the following `meta` object:

```js
{
  kind: 'list',
  type: type
}
```

## The dict combinator

You can define a dictionary type using the `dict(domain, codomain, name)` combinator where:

* `domain` is the type of keys
* `codomain` is the type of values
* `name`: is an optional string useful for debugging purposes

Instances of dicts are plain old JavaScript objects.

```js
const Tel = dict(Str, Num);

// costructor usage, `tel` is immutable
const tel = Tel({'jack': 4098, 'sape': 4139});
```

Dicts have the following `meta` object:

```js
{
  kind: 'dict',
  domain: domain,
  codomain: codomain
}
```

## The union combinator

You can define a union of types using the `union(types, name)` combinator where:

* `types` is a list of types
* `name` is an optional string useful for debugging purposes

```js
const ReactKey = union([Str, Num]);

ReactKey.is('a');  // => true
ReactKey.is(1);    // => true
ReactKey.is(true); // => false
```

Unions have the following `meta` object:

```js
{
  kind: 'union',
  types: types
}
```

### The dispatch function

In order to use a union as a constructor you must implement the static function:

```js
function (x: Any) -> Type
```

An example implementation for the `ReactKey` union:

```js
ReactKey.dispatch = function (x) {
  if (Str.is(x)) return Str;
  if (Num.is(x)) return Num;
};

// now you can do this without a fail
const key = ReactKey('a');
```

tcomb provides a default implementation of `dispatch` which you can override.

## The maybe combinator

In tcomb optional values of type `T` can be represented by `union([Nil, T])`. Since it's very common to handle optional values, tcomb provide an ad-hoc combinator.

You can define a maybe type using the `maybe(type, name)` combinator where:

* `type` is the wrapped type
* `name`: is an optional string useful for debugging purposes

```js
// the value of a radio input where null = no selection
const Radio = maybe(Str);

Radio.is('a');     // => true
Radio.is(null);    // => true
Radio.is(1);       // => false
```

Maybes have the following `meta` object:

```js
{
  kind: 'maybe',
  type: type
}
```

## The func combinator

Typed functions may be defined like this:

```js
// add takes two `Num`s and returns a `Num`
const add = func([Num, Num], Num)
    .of(function (x, y) { return x + y; });
```

And used like this:

```js
add("Hello", 2); // Raises error: Invalid `Hello` supplied to `Num`
add("Hello");    // Raises error: Invalid `Hello` supplied to `Num`

add(1, 2);       // Returns: 3
add(1)(2);       // Returns: 3
```

You can define a typed function using the `func(domain, codomain, name?)` combinator where:

* `domain` is the type of the function's argument (or list of types of the function's arguments)
* `codomain` is the type of the function's return value
* `name`: is an optional string useful for debugging purposes

Returns a function type whose functions have their domain and codomain specified and constrained.

`func` can be used to define function types using native types:

```js
// An `A` takes a `Str` and returns an `Num`
const A = func(Str, Num);
```

The domain and codomain can also be specified using types from any combinator including `func`:

```js
// A `B` takes a `Func` (which takes a `Str` and returns a `Num`) and returns a `Str`.
const B = func(func(Str, Num), Str);

// An `ExcitedStr` is a `Str` containing an exclamation mark
const ExcitedStr = subtype(Str, function (s) { return s.indexOf('!') !== -1; }, 'ExcitedStr');

// An `Exciter` takes a `Str` and returns an `ExcitedStr`
const Exciter = func(Str, ExcitedStr);
```

Additionally the domain can be expressed as a list of types:

```js
// A `C` takes an `A`, a `B` and a `Str` and returns a `Num`
const C = func([A, B, Str], Num);
```

Functions have the following `meta` object:

```js
{
  kind: 'func',
  domain: domain,
  codomain: codomain
}
```

### The `of(f: Function, curried?: boolean)` function

```js
func(A, B).of(f);
```

Returns a function where the domain and codomain are typechecked against the function type.

If the function is passed values which are outside of the domain or returns values which are outside of the codomain it will raise an error:

```js
const simpleQuestionator = Exciter.of(function (s) { return s + '?'; });
const simpleExciter      = Exciter.of(function (s) { return s + '!'; });

// Raises error:
// Invalid `Hello?` supplied to `ExcitedStr`, insert a valid value for the subtype
simpleQuestionator('Hello');

// Raises error: Invalid `1` supplied to `Str`
simpleExciter(1);

// Returns: 'Hello!'
simpleExciter('Hello');
```

The returned function may also be partially applied passing a `curried` additional param:

```js
// We can reasonably suggest that add has the following type signature
// add : Num -> Num -> Num
const add = func([Num, Num], Num)
    .of(function (x, y) { return x + y }, true);

const addHello = add("Hello"); // As this raises: "Error: Invalid `Hello` supplied to `Num`"

const add2 = add(2);
add2(1); // And this returns: 3
```

### The `is(x: any)` function

```js
func(A, B).is(x);
```

Returns `true` if x belongs to the type.

```js
Exciter.is(simpleExciter);      // Returns: true
Exciter.is(simpleQuestionator); // Returns: true

const id = function (x) { return x; };

func([Num, Num], Num).is(func([Num, Num], Num).of(id)); // Returns: true
func([Num, Num], Num).is(func(Num, Num).of(id));        // Returns: false
```

### Rules

1. Typed functions' domains are checked when they are called
2. Typed functions' codomains are checked when they return
3. The domain and codomain of a typed function's type is checked when the typed function is passed to a function type (such as when used as an argument in another typed function)

## Updating immutable instances

You can update an immutable instance with the `update` static function

```js
Type.update(instance, spec)
```

The following commands are compatible with the [Facebook Immutability Helpers](http://facebook.github.io/react/docs/update.html):

* $push
* $unshift
* $splice
* $set
* $apply
* $merge

Example:

```js
const p = new Point({x: 1, y: 2});

p = Point.update(p, {x: {'$set': 3}}); // => {x: 3, y: 2}
```

### Removing a value form a dict

```js
const Type = dict(Str, Num);
const instance = Type({a: 1, b: 2});
const updated = Type.update(instance, {$remove: ['a']}); // => {b: 2}
```

### Swapping two list elements

```js
const Type = list(Num);
const instance = Type([1, 2, 3, 4]);
const updated = Type.update(instance, {'$swap': {from: 1, to: 2}}); // => [1, 3, 2, 4]
```

### Adding other commands

You can add your custom commands updating the `t.update.commands` hash.

# Utils

There is a bunch of functions used internally by tcomb which are exported for convenience:

## fail(message: string): void

The default behaviour when failures occur is to throw a TypeError:

```js
function fail(message) {
  throw new TypeError(message);
}
```

You can override the default behaviour re-defining the `t.fail` function:

```js
t.fail = function (message) {
  // your code here
};
```

## update(instance: Object, spec: Object): Object

You can override the default behaviour re-defining the `t.update` function:

```js
t.update = function (instance, spec) {
  // your code here
};
```

## getTypeName(type: Type): string

Returns a type's name:

```js
getTypeName(Str); // => 'Str'
```

If name is not specified, fallbacks according to [http://flowtype.org](http://flowtype.org)

## mixin(target: Object, source: Object, override?: boolean): Object

Safe version of mixin, properties cannot be overwritten...

```js
mixin({a: 1}, {b: 2}); // => {a: 1, b: 2}
mixin({a: 1}, {a: 2}); // => throws
```

...unless `override = true`