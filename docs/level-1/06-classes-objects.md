# 06 · Classes & Objects Basics

## Defining a class

```dart
class Person {
  String name;
  int age;

  // Constructor
  Person(this.name, this.age);

  void introduce() {
    print("Hi, I'm $name and I'm $age years old.");
  }
}

void main() {
  var alice = Person('Alice', 30);
  alice.introduce();
  // Hi, I'm Alice and I'm 30 years old.
}
```

`Person(this.name, this.age)` is Dart's shorthand for "assign this constructor
argument directly to the field of the same name" — no need to write
`this.name = name;` by hand in the constructor body.

## Named constructors

A class can have multiple constructors as long as only one is unnamed.

```dart
class Point {
  double x, y;

  Point(this.x, this.y);

  // Named constructor -- an alternative way to build a Point
  Point.origin() : x = 0, y = 0;

  Point.fromList(List<double> coords) : x = coords[0], y = coords[1];

  @override
  String toString() => 'Point($x, $y)';
}

void main() {
  var p1 = Point(3, 4);
  var p2 = Point.origin();
  var p3 = Point.fromList([5, 6]);

  print(p1);   // Point(3.0, 4.0)
  print(p2);   // Point(0.0, 0.0)
  print(p3);   // Point(5.0, 6.0)
}
```

## Getters and setters

```dart
class Rectangle {
  double width;
  double height;

  Rectangle(this.width, this.height);

  // Computed getter -- looks like a field from the outside, but computed on access
  double get area => width * height;

  // Setter -- lets you write "rect.doubleWidth = true" style API if useful
  set scaleWidth(double factor) {
    width *= factor;
  }
}

void main() {
  var rect = Rectangle(4, 5);
  print(rect.area);   // 20.0

  rect.scaleWidth = 2;
  print(rect.area);   // 40.0
}
```

## Private members

Dart has no `private` keyword — instead, any identifier starting with an
underscore `_` is private to its own *library* (usually meaning its own file).

```dart
class BankAccount {
  double _balance;   // private -- not accessible outside this file

  BankAccount(this._balance);

  double get balance => _balance;

  void deposit(double amount) {
    if (amount <= 0) throw ArgumentError('Deposit must be positive');
    _balance += amount;
  }

  void withdraw(double amount) {
    if (amount > _balance) throw StateError('Insufficient funds');
    _balance -= amount;
  }
}

void main() {
  var account = BankAccount(100);
  account.deposit(50);
  account.withdraw(30);
  print(account.balance);   // 120.0
  // print(account._balance);   // fine within this same file, but would fail from another file
}
```

## Constructors with validation (initializer lists and asserts)

```dart
class Temperature {
  final double celsius;

  Temperature(this.celsius) : assert(celsius >= -273.15, 'Below absolute zero');

  double get fahrenheit => celsius * 9 / 5 + 32;
}

void main() {
  var temp = Temperature(25);
  print(temp.fahrenheit);   // 77.0
}
```

## Static members

`static` fields and methods belong to the class itself, not to any instance.

```dart
class MathUtils {
  static const double pi = 3.14159;

  static double circleArea(double radius) => pi * radius * radius;
}

void main() {
  print(MathUtils.pi);                 // 3.14159
  print(MathUtils.circleArea(3));      // 28.27431
}
```

## Overriding `toString`, `==`, and `hashCode`

```dart
class Point2 {
  final int x, y;
  Point2(this.x, this.y);

  @override
  String toString() => 'Point2($x, $y)';

  @override
  bool operator ==(Object other) =>
      other is Point2 && other.x == x && other.y == y;

  @override
  int get hashCode => Object.hash(x, y);
}

void main() {
  var a = Point2(1, 2);
  var b = Point2(1, 2);

  print(a);              // Point2(1, 2)
  print(a == b);         // true -- value equality, thanks to our override
  print(identical(a, b)); // false -- still two distinct objects in memory
}
```

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Constructor shorthand | `Person(this.name, this.age);` |
| Named constructor | `Person.guest() : name = 'Guest', age = 0;` |
| Getter | `double get area => width * height;` |
| Setter | `set scaleWidth(double f) { width *= f; }` |
| Private member | Prefix with `_`, e.g. `_balance` |
| Static member | `static const pi = 3.14159;` |

## Exercise

Design a `Book` class with private fields `_title`, `_author`, and `_pages`,
a standard constructor, a named constructor `Book.unknownAuthor(title, pages)`
that sets author to `"Unknown"`, a getter `summary` that returns a formatted
one-line description, and an override of `toString()`. Create a few `Book`
instances (some via each constructor) and print their summaries in a loop.
