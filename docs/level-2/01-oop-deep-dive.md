# 01 · OOP Deep Dive

Level 1 covered basic classes: fields, constructors, getters, and simple
inheritance with `extends`. Real Dart programs — and the Flutter framework
itself — lean heavily on three more tools for shaping object relationships:
**abstract classes** (define a contract with some shared code), **mixins**
(share behavior across unrelated class hierarchies), and **interfaces**
(Dart has no `interface` keyword — every class *is* an interface). Knowing
when to reach for each one is what separates "code that compiles" from code
that's actually maintainable as it grows.

## Abstract classes

An abstract class can't be instantiated directly. It exists to define a
shared contract — some methods it leaves unimplemented (`abstract`) and,
unlike a pure interface, it can also hold real fields and concrete methods
that subclasses inherit for free.

```dart
abstract class Shape {
  // Abstract classes can hold real state and concrete methods...
  final String label;
  Shape(this.label);

  // ...but a method with no body is abstract: subclasses must implement it.
  double area();

  // Concrete methods can call the abstract ones -- polymorphism at work.
  void describe() {
    print('$label has area ${area().toStringAsFixed(2)}');
  }
}

class Circle extends Shape {
  final double radius;
  Circle(this.radius) : super('Circle');

  @override
  double area() => 3.14159 * radius * radius;
}

class Square extends Shape {
  final double side;
  Square(this.side) : super('Square');

  @override
  double area() => side * side;
}

void main() {
  final shapes = <Shape>[Circle(2), Square(3)];
  for (final shape in shapes) {
    shape.describe();
  }
  // Circle has area 12.57
  // Square has area 9.00

  // Shape s = Shape('x'); // Error: abstract classes can't be instantiated directly
}
```

Use an abstract class when subclasses share **both** a contract and some
common implementation (here, `describe()` is written once and reused by
every shape).

## Interfaces — every class is one for free

Dart has no separate `interface` keyword. Instead, *every* class implicitly
defines an interface consisting of its public members. Any class can commit
to that shape with `implements`, but — this is the key trap for people
coming from Java or C# — `implements` copies **zero** implementation. Every
member has to be rewritten from scratch, even ones that had a body in the
original class.

```dart
class JsonSerializable {
  Map<String, dynamic> toJson() => {};
}

class User implements JsonSerializable {
  final String name;
  final int age;
  User(this.name, this.age);

  @override
  Map<String, dynamic> toJson() => {'name': name, 'age': age};
}

void main() {
  final user = User('Ada', 30);
  print(user.toJson());   // {name: Ada, age: 30}
}
```

If `JsonSerializable.toJson()` had real logic you wanted to reuse, `implements`
would be the wrong tool — you'd want `extends` (single inheritance) or a
mixin (see below).

| Keyword | Inherits implementation? | How many can you use? |
|---|---|---|
| `extends` | Yes, from one superclass | One |
| `implements` | No — contract only | Many |
| `with` (mixin) | Yes, from each mixin | Many |

## Mixins — shared behavior without a shared ancestor

A mixin packages up behavior that unrelated classes can "mix in" using
`with`, without forcing them into a single-inheritance chain. It's Dart's
answer to "I want to share this method across a `Duck` and a `Robot`, but
they have nothing else in common."

```dart
mixin Flyer {
  void fly() => print('Flying through the air');
}

mixin Swimmer {
  void swim() => print('Swimming through water');
}

// A class can mix in multiple behaviors without inheriting from a shared base.
class Duck with Flyer, Swimmer {
  void quack() => print('Quack!');
}

void main() {
  final duck = Duck();
  duck.fly();     // Flying through the air
  duck.quack();   // Quack!
  duck.swim();    // Swimming through water
}
```

### The trap: mixin order determines method resolution

If two mixed-in mixins define the *same* method, Dart doesn't error — it
resolves the conflict by linearizing the mixins right-to-left, so **the
rightmost mixin in the `with` clause wins**, because it sits closest to the
class in the resulting chain.

```dart
mixin LoudLogger {
  void log(String msg) => print('[LOUD] ${msg.toUpperCase()}');
}

mixin QuietLogger {
  void log(String msg) => print('[quiet] $msg');
}

class ServiceA with LoudLogger, QuietLogger {}
class ServiceB with QuietLogger, LoudLogger {}

void main() {
  ServiceA().log('starting up');   // [quiet] starting up  -- QuietLogger is rightmost
  ServiceB().log('starting up');   // [LOUD] STARTING UP   -- LoudLogger is rightmost
}
```

This is a genuinely common source of "why did my override not take effect"
bugs — if two mixins collide, swapping their order in the `with` clause
silently changes behavior with no warning from the analyzer.

### Restricting a mixin with `on`

A mixin can require that it only be used on classes that already have a
particular supertype, using `on`. This lets the mixin safely call members it
knows must exist, without duck-typing risk.

```dart
abstract class Animal {
  String get name;
}

mixin Hunter on Animal {
  void hunt() => print('$name is hunting');
}

class Lion extends Animal with Hunter {
  @override
  final String name;
  Lion(this.name);
}

void main() {
  final lion = Lion('Leo');
  lion.hunt();   // Leo is hunting

  // class Bird with Hunter {} // Error: Bird doesn't extend/implement Animal
}
```

## Choosing between them

| Need | Reach for |
|---|---|
| Share both state and behavior down a natural "is-a" hierarchy | `abstract class` + `extends` |
| Guarantee a class has certain methods, regardless of ancestry | `implements` |
| Share reusable behavior across otherwise-unrelated classes | `mixin` + `with` |
| Share behavior, but only for classes that already have specific members | `mixin ... on SomeType` |

A single class can combine all three: `class Lion extends Animal with Hunter
implements Comparable<Lion> { ... }` — extend one base, mix in one or more
behaviors, and commit to one or more interfaces.

## Exercise

Model a small plugin system: an abstract class `Plugin` with an abstract
method `String get name` and a concrete method `void log(String msg)` that
prints `'[$name] $msg'`. Create two mixins, `Cacheable` (adds a `Map<String,
dynamic> cache = {}` field and a `void remember(String key, dynamic value)`
method) and `Loggable` (adds a `void trace(String msg)` that calls `log` —
constrain it with `on Plugin` so it can). Build a class `ApiPlugin extends
Plugin with Cacheable, Loggable` that implements `name`, and in `main()`
create an instance, call `remember`, `trace`, and print the cache contents.
