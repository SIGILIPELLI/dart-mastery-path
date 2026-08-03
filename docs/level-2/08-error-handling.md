# 08 · Error Handling

[Async basics](../level-1/08-async-basics.md) showed a quick `try`/`catch`
around an `await`. This module covers the full picture: designing your own
exception types, handling multiple failure kinds in the right order,
`finally`, `rethrow`, and — critically for anything built on `Future`s or
`Stream`s — how errors propagate (or silently vanish) in async code.

## `Exception` vs `Error`

Dart's throwable hierarchy distinguishes two intents. **`Exception`**
(implement it, or throw a built-in like `FormatException`) signals an
*expected*, recoverable failure — bad input, a network timeout, a missing
file. **`Error`** (like `StateError`, `ArgumentError`, `RangeError`) signals
a *programmer mistake* — something that should have been prevented by
correct code, not routinely caught and handled.

```dart
class InsufficientFundsException implements Exception {
  final double requested;
  final double available;
  InsufficientFundsException(this.requested, this.available);

  @override
  String toString() =>
      'InsufficientFundsException: requested \$$requested but only \$$available available';
}

class Account {
  double _balance;
  Account(this._balance);

  void withdraw(double amount) {
    if (amount > _balance) {
      throw InsufficientFundsException(amount, _balance);
    }
    _balance -= amount;
  }
}

void main() {
  final account = Account(100);

  try {
    account.withdraw(150);
  } on InsufficientFundsException catch (e) {
    print('Handled: $e');
    // Handled: InsufficientFundsException: requested $150.0 but only $100.0 available
  }
}
```

`implements Exception` is just a marker — it doesn't require any particular
members — but using it (rather than throwing a plain `String` or `Object`)
communicates intent clearly and lets callers catch it specifically with
`on InsufficientFundsException`.

## `finally` always runs

```dart
void main() {
  final account = Account(100);

  try {
    account.withdraw(30);
    print('Withdrawal succeeded');
  } on InsufficientFundsException catch (e) {
    print('Handled: $e');
  } finally {
    print('Transaction attempt complete');
  }
  // Withdrawal succeeded
  // Transaction attempt complete
}
```

`finally` runs whether the `try` block succeeds, throws and gets caught, or
throws and *isn't* caught (in which case it runs before the exception keeps
propagating). It's the right place for cleanup — closing a file, releasing a
lock — that must happen regardless of outcome.

## Multiple catch clauses — order matters

```dart
int parseAndDouble(String input) {
  final n = int.parse(input); // throws FormatException on bad input
  return n * 2;
}

void demoMultiCatch(String input) {
  try {
    print(parseAndDouble(input));
  } on FormatException catch (e) {
    // More specific catches must come BEFORE more general ones --
    // order matters, same as in most languages.
    print('Not a number: ${e.message}');
  } on Exception catch (e) {
    print('Some other exception: $e');
  } catch (e) {
    // Bare `catch` also catches Errors (bugs), not just Exceptions (failures).
    print('Unknown throwable: $e');
  }
}

void main() {
  demoMultiCatch('42');    // 84
  demoMultiCatch('abc');   // Not a number: Invalid radix-10 number
}
```

Dart checks `on` clauses top-to-bottom and uses the first match — a bare
`catch (e)` at the end acts as a catch-all, matching anything not already
caught above it, including `Error`s. Put it last, and put the most specific
types first.

## `rethrow` — preserve the original stack trace

```dart
void logAndRethrow() {
  try {
    throw StateError('deep failure');
  } catch (e) {
    print('Logging before rethrow: $e');
    rethrow; // preserves the original stack trace, unlike `throw e;`
  }
}

void main() {
  try {
    logAndRethrow();
  } catch (e) {
    print('Caught after rethrow: $e');
  }
  // Logging before rethrow: Bad state: deep failure
  // Caught after rethrow: Bad state: deep failure
}
```

Use `rethrow` (not `throw e`) whenever you want to log or react to an error
partway up the call stack without swallowing it — `throw e` technically
works but resets the stack trace to this new throw site, making the
original failure harder to trace back.

## Errors and `async`/`await`

An error thrown inside an `async` function propagates through `await`
exactly like a synchronous exception — a `try`/`catch` around the `await`
catches it normally.

```dart
Future<void> asyncFailure() async {
  await Future.delayed(const Duration(milliseconds: 10));
  throw Exception('async boom');
}

Future<void> main() async {
  try {
    await asyncFailure();
  } catch (e) {
    print('Caught async error: $e');
    // Caught async error: Exception: async boom
  }
}
```

### The trap: an un-awaited `Future`'s error goes nowhere useful

```dart
Future<void> main() async {
  asyncFailure(); // fire-and-forget -- NOT awaited
  print('main continues');
  await Future.delayed(const Duration(milliseconds: 100));
}
```

```
main continues
Unhandled exception:
Exception: async boom
#0      asyncFailure (file:...)
<asynchronous suspension>
```

Because the `Future` returned by `asyncFailure()` was never `await`ed (or
given a `.catchError()`), your surrounding `try`/`catch` has nothing to
attach to — the error surfaces later as an **unhandled exception**, often
crashing the isolate, with no connection in the stack trace back to where it
was called from. This is one of the most common real-world Dart/Flutter bugs:
a call that *looks* handled because it's inside a `try` block, but isn't,
because the `await` was missing. Rule of thumb: any `Future`-returning call
you don't `await` needs an explicit `.catchError()`, or a clear comment
explaining why its failure is safe to ignore.

## Cheat sheet

| Tool | Use |
|---|---|
| `implements Exception` | Mark a custom type as an expected, recoverable failure |
| `on SomeType catch (e)` | Catch only a specific throwable type |
| `catch (e)` (bare) | Catch anything, including `Error`s — put it last |
| `finally` | Runs regardless of success/failure/rethrow |
| `rethrow` | Re-throw the current exception, preserving its original stack trace |
| `await` inside `try`/`catch` | Catches errors from the awaited `Future` normally |
| Un-awaited `Future` | Its error bypasses any surrounding `try`/`catch` |

## Exercise

Define an exception hierarchy for a simple file-parsing tool: a base
`ParseException implements Exception` with a `String reason` field, and two
subclasses, `EmptyFileException` and `InvalidFormatException`, each calling
the base constructor with a specific reason. Write a function
`Map<String, dynamic> parseConfig(String contents)` that throws
`EmptyFileException` if `contents.trim()` is empty, `InvalidFormatException`
if it doesn't contain a `:` character, and otherwise splits on the first `:`
and returns a one-entry map. Call it with three inputs (empty, malformed,
valid) inside a `try` with `on EmptyFileException`, `on
InvalidFormatException`, and a final bare `catch`, confirming each takes the
right branch.
