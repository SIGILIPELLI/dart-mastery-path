# 06 · Testing Advanced

[Level 2's testing module](../level-2/04-testing.md) covered the basics —
`test()`, `expect()`, and simple matchers. Real test suites also need to
isolate code from time, randomness, and slow dependencies, and to assert on
asynchronous behavior correctly. This module covers `setUp`/`group`, hand-
rolled fakes (no mocking framework needed for most cases), and the async
matcher traps that catch people out.

## `group` and `setUp`: shared fixtures without duplication

`group()` names a related batch of tests; `setUp()` runs before **each**
test in its group, giving every test a fresh fixture instead of a shared,
possibly-mutated one.

```dart
import 'package:test/test.dart';

abstract class Clock {
  DateTime now();
}

class FakeClock implements Clock {
  FakeClock(this._time);
  DateTime _time;
  @override
  DateTime now() => _time;
  void advance(Duration d) => _time = _time.add(d);
}

class Session {
  Session(this._clock, {Duration timeout = const Duration(minutes: 5)})
      : _timeout = timeout,
        _started = _clock.now();

  final Clock _clock;
  final Duration _timeout;
  final DateTime _started;

  bool get isExpired => _clock.now().difference(_started) > _timeout;
}

void main() {
  group('Session with a fake clock', () {
    late FakeClock clock;

    setUp(() {
      clock = FakeClock(DateTime(2024, 1, 1)); // fresh clock for every test
    });

    test('is not expired immediately', () {
      final session = Session(clock);
      expect(session.isExpired, isFalse);
    });

    test('expires after the timeout elapses', () {
      final session = Session(clock, timeout: const Duration(minutes: 1));
      clock.advance(const Duration(minutes: 2));
      expect(session.isExpired, isTrue);
    });
  });
}
```

```
00:00 +2: All tests passed!
```

The `Clock` interface is the real trick here: `Session` never calls
`DateTime.now()` directly, so tests control time completely instead of
`sleep`-ing real seconds to observe a timeout.

## Fakes over mocking frameworks

A hand-written fake implementing an interface is often clearer than a
mocking library, and needs no extra dependency:

```dart
abstract class Repository {
  Future<String> fetchName(int id);
}

class FakeRepository implements Repository {
  final Map<int, String> data = {1: 'Ada'};
  int callCount = 0;

  @override
  Future<String> fetchName(int id) async {
    callCount++;
    final name = data[id];
    if (name == null) throw StateError('not found: $id');
    return name;
  }
}

void main() {
  group('Repository fake', () {
    test('returns a known name', () async {
      final repo = FakeRepository();
      final name = await repo.fetchName(1);
      expect(name, equals('Ada'));
      expect(repo.callCount, 1); // fakes can also assert on interactions
    });

    test('throws for an unknown id', () {
      final repo = FakeRepository();
      expect(() => repo.fetchName(2), throwsA(isA<StateError>()));
    });
  });
}
```

`repo.callCount` shows the other advantage of a fake over a "pure" test
double: it can track how it was used and let a test assert on that, exactly
like a mock's verify step, with no extra API to learn.

## Async matchers: `completion` and `throwsA`

`expect()` can take a `Future` directly if the matcher understands async —
`completion()` unwraps a `Future`'s success value; `throwsA()` (used above)
unwraps its failure.

```dart
test('async matcher: completes with a value', () {
  Future<int> compute() async {
    await Future.delayed(const Duration(milliseconds: 10));
    return 42;
  }

  expect(compute(), completion(equals(42)));
});
```

`test()` itself doesn't need to be `async` here — `completion()` returns a
matcher that the test runner awaits internally, and it registers the
expectation as part of the test's async work so the test doesn't finish
(and pass falsely) before the future resolves.

## The trap: `throwsA` needs a closure, not a called function

`throwsA` matches an exception thrown **while `expect` evaluates its
matcher** — which only works if the thing under test hasn't already run (and
thrown) before `expect` is even called.

```dart
int boom() => throw StateError('bad');

void main() {
  test('trap: calling eagerly throws before expect runs', () {
    expect(boom(), throwsA(isA<StateError>()));
  });
}
```

```
trap: calling eagerly throws before expect runs [E]
  Bad state: bad
  test/trap_test.dart 3:15  boom
  test/trap_test.dart 7:12  main.<fn>
```

`boom()` throws while Dart evaluates the argument list, before `expect` is
ever entered — the test fails with an *uncaught* exception rather than a
matcher mismatch. The fix is always to pass a closure so `expect` controls
when the call happens: `expect(() => boom(), throwsA(isA<StateError>()))`.
This is exactly the same shape of mistake as `FakeRepository`'s working
`expect(() => repo.fetchName(2), throwsA(...))` above — note the `() =>`.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `group(name, () {...})` | Namespace and organize related tests |
| `setUp(() {...})` | Runs before every test in the enclosing group — fresh fixtures |
| `tearDown(() {...})` | Runs after every test — release resources |
| Fake implementing an interface | Test double with real (simplified) behavior, no framework needed |
| `completion(matcher)` | Assert on the value a `Future` completes with |
| `throwsA(matcher)` | Assert an exception is thrown — needs `() => ...`, not a called value |
| `isA<T>()` | Matcher: value is an instance of `T` |

## Exercise

Write a `RetryingClient` class that takes a `Future<String> Function()`
fetcher and a `Clock`, and retries the fetcher up to 3 times with a 1-second
fake-clock delay between attempts if it throws, returning the first
successful result or rethrowing after the third failure. Test it with a
`group` that uses `setUp` to build a fetcher-call counter, covering: succeeds
on the first try, succeeds on the third try after two failures, and
exhausts all retries and rethrows. Use `throwsA` correctly (with a closure)
for the failing case.
