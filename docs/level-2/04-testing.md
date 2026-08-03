# 04 · Testing with the test package

Every lesson so far has been verified by eyeballing printed output. Real
projects instead write automated tests: code that checks your code, runs in
seconds, and catches regressions the moment you introduce them. Dart's
official `package:test` is the standard tool for this, and it's simple
enough to be worth using from day one of any real project.

## Setting up

Add `test` as a dev dependency (it's only needed while developing, not at
runtime) and create a `test/` directory next to `lib/`:

```bash
dart pub add --dev test
```

```
my_project/
├── lib/
│   └── calculator.dart
├── test/
│   └── calculator_test.dart
└── pubspec.yaml
```

## The code under test

```dart
// lib/calculator.dart
class Calculator {
  int add(int a, int b) => a + b;

  int divide(int a, int b) {
    if (b == 0) throw ArgumentError('Cannot divide by zero');
    return a ~/ b;
  }
}
```

## Writing tests

```dart
// test/calculator_test.dart
import 'package:test/test.dart';
import 'package:my_project/calculator.dart';

void main() {
  group('Calculator', () {
    late Calculator calc;

    // setUp runs before EVERY test in this group -- fresh state each time.
    setUp(() {
      calc = Calculator();
    });

    test('adds two positive numbers', () {
      expect(calc.add(2, 3), equals(5));
    });

    test('divides evenly', () {
      expect(calc.divide(10, 2), 5);
    });

    test('throws on division by zero', () {
      expect(() => calc.divide(10, 0), throwsArgumentError);
    });
  });
}
```

Run the whole suite with:

```bash
dart test
```

```
00:00 +0: loading test/calculator_test.dart
00:00 +1: Calculator adds two positive numbers
00:00 +2: Calculator divides evenly
00:00 +3: Calculator throws on division by zero
00:00 +3: All tests passed!
```

## Reading a failure

Tests are only useful if failures are easy to diagnose. Here's what it looks
like when an assertion doesn't hold — note the exact expected/actual values
package:test prints:

```dart
test('intentionally wrong expectation', () {
  final calc = Calculator();
  expect(calc.add(2, 2), equals(5));
});
```

```
00:00 +0 -1: intentionally wrong expectation [E]
  Expected: <5>
    Actual: <4>

  package:matcher          expect
  test/fail_test.dart 7:5  main.<fn>

00:00 +0 -1: Some tests failed.
```

`+0 -1` means zero passed, one failed — that count updates live as the suite
runs, which is handy for spotting a hang or a slow test in a big suite.

## `group`, `setUp`, and `tearDown`

`group` organizes related tests and gives them a shared label in output.
`setUp` runs before each test in its group (and any nested groups);
`tearDown` runs after each one, even if the test failed — the place to
release resources like open files or database connections.

```dart
import 'package:test/test.dart';

void main() {
  group('Counter', () {
    late int counter;

    setUp(() {
      counter = 0; // fresh state for every single test
      print('setUp: counter reset');
    });

    tearDown(() {
      print('tearDown: cleaning up');
    });

    test('starts at zero', () {
      expect(counter, 0);
    });

    test('can be incremented', () {
      counter++;
      expect(counter, 1);
    });
  });
}
```

Each test gets its own `setUp`/`tearDown` pair — tests never leak state into
one another, which is exactly what makes them trustworthy in any order.

## Useful matchers

`expect(actual, matcher)` is the core assertion. Beyond plain equality,
`package:test` ships a library of expressive matchers:

| Matcher | Checks |
|---|---|
| `equals(x)` | Deep equality (same as the bare value for most types) |
| `throwsArgumentError` | The callback throws an `ArgumentError` |
| `throwsA(isA<MyException>())` | The callback throws a specific custom type |
| `isA<T>()` | The value's runtime type is (or extends) `T` |
| `contains(x)` | A `List`/`String`/`Map` contains `x` |
| `isEmpty` / `isNotEmpty` | Collection or string emptiness |
| `closeTo(value, delta)` | A `double` is within `delta` of `value` |

```dart
import 'package:test/test.dart';

class InvalidTemperatureException implements Exception {
  final String message;
  InvalidTemperatureException(this.message);
}

double toFahrenheit(double celsius) {
  if (celsius < -273.15) {
    throw InvalidTemperatureException('Below absolute zero');
  }
  return celsius * 9 / 5 + 32;
}

void main() {
  test('converts a normal temperature', () {
    expect(toFahrenheit(0), closeTo(32.0, 0.001));
  });

  test('rejects an impossible temperature', () {
    expect(
      () => toFahrenheit(-300),
      throwsA(isA<InvalidTemperatureException>()),
    );
  });
}
```

## Testing async code

`test()` accepts an `async` callback, and `expectLater` awaits a matcher
against a `Future` or `Stream` — no special ceremony needed beyond the
`async`/`await` you already know from [Async basics](../level-1/08-async-basics.md).

```dart
import 'package:test/test.dart';

Future<String> fetchGreeting() async {
  await Future.delayed(const Duration(milliseconds: 50));
  return 'Hello!';
}

void main() {
  test('fetchGreeting resolves with the right value', () async {
    final result = await fetchGreeting();
    expect(result, 'Hello!');
  });

  test('fetchGreeting completes as a Future<String>', () {
    expect(fetchGreeting(), completion('Hello!'));
  });
}
```

## Exercise

Create a small `lib/string_utils.dart` with a function `String
truncate(String input, int maxLength)` that returns `input` unchanged if
it's `maxLength` or shorter, and otherwise returns the first `maxLength`
characters followed by `'...'`. Write `test/string_utils_test.dart` with a
`group` covering at least four cases: a string shorter than the limit, one
exactly at the limit, one longer than the limit, and an empty string. Run
`dart test` and confirm all cases pass; then temporarily break the function
(e.g. off-by-one on the length check) and confirm the test output clearly
shows which case failed and why.
