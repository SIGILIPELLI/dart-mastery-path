# 04 · Design Patterns in Dart

[Classes & objects](../level-1/06-classes-objects.md) and
[generics](../level-2/06-generics.md) gave you the building blocks. Design
patterns are named solutions to recurring structural problems — this module
covers four that show up constantly in Dart/Flutter codebases: Singleton,
Factory, Observer, and Strategy.

## Singleton: exactly one instance

A private constructor plus a `factory` constructor that always returns the
same cached instance.

```dart
class AppConfig {
  AppConfig._internal(this.apiUrl); // private -- can't be called from outside
  static final AppConfig _instance = AppConfig._internal('https://api.example.com');
  factory AppConfig() => _instance;
  final String apiUrl;
}

void main() {
  final a = AppConfig();
  final b = AppConfig();
  print('Same instance: ${identical(a, b)}');
}
// Same instance: true
```

`factory` constructors are Dart's built-in escape hatch for "don't
necessarily create a new instance" — a regular constructor is required to
allocate a new object, but a `factory` can return a cached one, an instance
of a subclass, or `null` (for a nullable return type).

## Factory: choosing the concrete type at runtime

`Shape.fromType(...)` decides which concrete class to instantiate based on a
runtime value, so calling code only ever depends on the abstract `Shape`
interface.

```dart
abstract class Shape {
  double area();
  factory Shape.fromType(String type, double a, [double b = 0]) {
    switch (type) {
      case 'circle':
        return Circle(a);
      case 'rectangle':
        return Rectangle(a, b);
      default:
        throw ArgumentError('Unknown shape: $type');
    }
  }
}

class Circle implements Shape {
  Circle(this.radius);
  final double radius;
  @override
  double area() => 3.14159 * radius * radius;
}

class Rectangle implements Shape {
  Rectangle(this.width, this.height);
  final double width;
  final double height;
  @override
  double area() => width * height;
}

void main() {
  final shapes = [Shape.fromType('circle', 2), Shape.fromType('rectangle', 3, 4)];
  for (final s in shapes) {
    print('${s.runtimeType} area: ${s.area()}');
  }
}
// Circle area: 12.56636
// Rectangle area: 12.0
```

Adding a new shape type only requires a new class plus one more `case` in
the factory — every caller that already depends on `Shape` needs no changes.

## Observer: broadcasting state changes

An object (the "subject") keeps a list of listeners and notifies all of
them whenever its state changes. This is the pattern behind `ChangeNotifier`
in Flutter, `Stream` broadcasting, and any pub/sub system.

```dart
abstract class Observer {
  void onChanged(int value);
}

class Counter {
  int _value = 0;
  final List<Observer> _observers = [];

  void subscribe(Observer o) => _observers.add(o);

  void increment() {
    _value++;
    for (final o in _observers) {
      o.onChanged(_value);
    }
  }
}

class LoggingObserver implements Observer {
  @override
  void onChanged(int value) => print('Logger: value is now $value');
}

void main() {
  final counter = Counter();
  counter.subscribe(LoggingObserver());
  counter.increment();
  counter.increment();
}
// Logger: value is now 1
// Logger: value is now 2
```

## Strategy: swapping behavior at runtime

An interchangeable algorithm, held as a field so it can change after
construction — here, how a `Checkout` computes its total.

```dart
abstract class DiscountStrategy {
  double apply(double price);
}

class NoDiscount implements DiscountStrategy {
  @override
  double apply(double price) => price;
}

class PercentOff implements DiscountStrategy {
  PercentOff(this.percent);
  final double percent;
  @override
  double apply(double price) => price * (1 - percent / 100);
}

class Checkout {
  Checkout(this.strategy);
  DiscountStrategy strategy;
  double total(double price) => strategy.apply(price);
}

void main() {
  final checkout = Checkout(NoDiscount());
  print('No discount: ${checkout.total(100)}');
  checkout.strategy = PercentOff(20);
  print('20% off: ${checkout.total(100)}');
}
// No discount: 100.0
// 20% off: 80.0
```

## The trap: `identical()` vs `==` for singleton checks

Verifying a singleton with `a == b` instead of `identical(a, b)` is a common
mistake — if the class ever overrides `==` (or mixes in something that
does), two *different* instances could compare equal, hiding a broken
singleton (e.g. one where the private constructor got called directly via
reflection, or a refactor accidentally removed the `factory`). `identical()`
checks object identity regardless of any `==` override, which is the actual
property a singleton needs to guarantee.

## Cheat sheet

| Pattern | Problem it solves | Dart mechanism |
|---|---|---|
| Singleton | Exactly one shared instance | Private constructor + `factory` returning a cached field |
| Factory | Pick concrete type at runtime, hide it from callers | `factory` constructor with a `switch`/lookup |
| Observer | Broadcast state changes to many listeners | List of interfaces, loop-and-notify on mutation |
| Strategy | Swap an algorithm without changing the caller | Interface held as a mutable field |
| `identical(a, b)` | True object-identity check | Ignores any `==` override |

## Exercise

Implement the Strategy pattern for a `Logger` class with a mutable
`LogFormatter` field. Write two formatters — `PlainFormatter` (returns the
message unchanged) and `JsonFormatter` (returns
`{"message": "$msg", "level": "$level"}`) — and a `Logger.log(String level,
String message)` that prints `formatter.format(level, message)`. Then add a
Factory constructor `LogFormatter.fromName(String name)` that returns the
right formatter for `'plain'` or `'json'`, throwing `ArgumentError` for
anything else, and use it to build a `Logger` from a string read at
"runtime" (a hardcoded variable is fine).
