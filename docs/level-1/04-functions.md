# 04 · Functions

## Basic function syntax

```dart
int add(int a, int b) {
  return a + b;
}

void main() {
  print(add(2, 3));   // 5
}
```

## Arrow syntax for one-expression bodies

If a function body is a single expression, you can skip `{ return ...; }` and
use `=>` instead:

```dart
int square(int x) => x * x;

void main() {
  print(square(5));   // 25
}
```

## Optional positional parameters

Wrap parameters in `[ ]` to make them optional; give them a default with `=`.

```dart
String greet(String name, [String greeting = 'Hello']) {
  return '$greeting, $name!';
}

void main() {
  print(greet('Ada'));                // Hello, Ada!
  print(greet('Ada', 'Welcome'));      // Welcome, Ada!
}
```

## Named parameters

Wrap parameters in `{ }` to make them named at the call site — great for
functions with many parameters, since order no longer matters and calls
become self-documenting. Named parameters are optional by default unless
marked `required`.

```dart
void createUser({required String name, int age = 18, String? email}) {
  print('$name, age $age, email: ${email ?? "none"}');
}

void main() {
  createUser(name: 'Grace');
  // Grace, age 18, email: none

  createUser(name: 'Linus', age: 34, email: 'linus@example.com');
  // Linus, age 34, email: linus@example.com
}
```

You can mix positional and named parameters, but not positional-optional (`[]`)
and named (`{}`) in the same parameter list.

## Functions as first-class values

Functions can be assigned to variables, passed as arguments, and returned from
other functions — Dart treats them as regular values.

```dart
void main() {
  // Assign a function to a variable
  int Function(int, int) multiply = (a, b) => a * b;
  print(multiply(4, 5));   // 20

  // Pass a function as an argument
  List<int> numbers = [1, 2, 3, 4, 5];
  var doubled = numbers.map((n) => n * 2).toList();
  print(doubled);   // [2, 4, 6, 8, 10]

  // A function that returns a function (closure)
  print(makeMultiplier(3)(7));   // 21
}

Function makeMultiplier(int factor) {
  return (int value) => value * factor;
}
```

## Anonymous functions (lambdas)

```dart
void main() {
  var numbers = [1, 2, 3, 4, 5];

  // Anonymous function passed directly
  numbers.forEach((n) {
    print('Value: $n');
  });

  // Shorter arrow-syntax anonymous function
  var squares = numbers.map((n) => n * n).toList();
  print(squares);   // [1, 4, 9, 16, 25]
}
```

## Closures

A closure captures variables from its surrounding scope, keeping them alive
even after the enclosing function returns.

```dart
Function makeCounter() {
  int count = 0;
  return () {
    count++;
    return count;
  };
}

void main() {
  var counter = makeCounter();
  print(counter());   // 1
  print(counter());   // 2
  print(counter());   // 3

  var anotherCounter = makeCounter();
  print(anotherCounter());   // 1 -- independent state, its own `count`
}
```

## Recursion

```dart
int factorial(int n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}

void main() {
  print(factorial(5));   // 120
}
```

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `int add(int a, int b) { ... }` | Standard function with a block body |
| `int square(int x) => x * x;` | Arrow syntax for single-expression bodies |
| `[String greeting = 'Hi']` | Optional positional parameter with default |
| `{required String name}` | Required named parameter |
| `{int age = 18}` | Optional named parameter with default |
| `int Function(int, int)` | Type of a function taking two ints, returning an int |

## Exercise

Write a function `describeBox` that takes required named parameters `width`
and `height` (both `double`) and an optional named parameter `label` (default
`"Box"`), and returns a formatted string with the label, dimensions, and
computed area. Then write a higher-order function `applyTwice` that takes an
`int Function(int)` and an `int` value, and returns the result of applying the
function to the value twice (e.g. `applyTwice((x) => x + 3, 10)` should return
`16`).
