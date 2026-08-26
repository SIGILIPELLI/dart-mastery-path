# 01 · Advanced Async Patterns

[Async basics](../level-1/08-async-basics.md) and
[streams](../level-2/03-streams.md) covered `Future`, `async`/`await`, and
`Stream`. This module covers the patterns that show up once a codebase has
more than one async operation happening at a time: running work
concurrently, bridging callback APIs into `Future`s, and the sharpest of
Dart's async traps — a missing `await` that turns a normal exception into an
unhandled crash.

## Running futures concurrently with `Future.wait`

`await`ing futures one after another runs them **sequentially**. `Future
.wait` starts them all immediately and waits for every one to finish —
critical for anything that's actually independent (parallel API calls,
parallel file reads).

```dart
Future<int> slow(int id, int ms) async {
  await Future.delayed(Duration(milliseconds: ms));
  return id;
}

Future<void> main() async {
  final sw = Stopwatch()..start();
  final results = await Future.wait([slow(1, 100), slow(2, 100), slow(3, 100)]);
  print('Future.wait results: $results in ${sw.elapsedMilliseconds}ms (concurrent)');
}
// Future.wait results: [1, 2, 3] in 105ms (concurrent)
```

Three 100ms delays finish in ~105ms, not ~300ms — they ran side by side.
Results come back in the **same order the futures were listed**, regardless
of which one actually finished first.

## Bridging callback APIs with `Completer`

Some APIs (timers, native callbacks, older event-based libraries) hand you
a value through a callback instead of a `Future`. `Completer<T>` lets you
wrap one in a `Future` so it composes with the rest of your `async` code.

```dart
import 'dart:async';

Future<void> main() async {
  final completer = Completer<String>();
  Timer(const Duration(milliseconds: 50), () => completer.complete('done via Completer'));
  print(await completer.future);
}
// done via Completer
```

Call `.complete()` (or `.completeError()`) exactly once — a second call on
an already-completed `Completer` throws `Bad state: Future already
completed`. This is a common bug in code paths with more than one possible
"finish" branch (e.g. both a success callback and a timeout firing).

## `eagerError`: when one failure should stop everything

By default, `Future.wait` completes with the **first** error it sees,
without waiting for the rest — `eagerError` only changes what happens to
errors *after* that point.

```dart
Future<int> maybeFail(int id, bool fail) async {
  if (fail) throw Exception('failure $id');
  return id;
}

Future<void> main() async {
  try {
    await Future.wait([maybeFail(1, false), maybeFail(2, true), maybeFail(3, false)]);
  } catch (e) {
    print('Future.wait (default) stops at first error: $e');
  }
}
// Future.wait (default) stops at first error: Exception: failure 2
```

Either way, `Future.wait` internally keeps listening to every future you
gave it (so a later failure among the others never becomes an unhandled
error behind your back) — the `eagerError` flag only controls whether *your*
`await` returns the instant the first error shows up, or waits for all of
them to settle before reporting.

## The trap: a missing `await` turns a caught exception into a crash

This is the single most dangerous async mistake in Dart. An `async`
function that throws behaves completely differently depending on whether
its caller awaits it.

```dart
Future<void> risky() async {
  await Future.delayed(const Duration(milliseconds: 10));
  throw Exception('boom from risky');
}

Future<void> main() async {
  try {
    risky(); // missing await!
    print('main: called risky (no await), moving on');
  } catch (e) {
    print('this never runs: $e');
  }

  await Future.delayed(const Duration(milliseconds: 50));
  print('main: done');
}
```

```
main: called risky (no await), moving on
Unhandled exception:
Exception: boom from risky
#0      risky (...)
```

The `try`/`catch` around `risky()` does nothing, because `risky()` returns
immediately with a pending `Future` — the function's body (and its eventual
throw) runs later, completely outside that `try` block's stack frame. By the
time the exception actually happens, there's no catcher listening, so it
crashes the isolate instead. The fix is always `await risky();` — and this
is exactly the class of bug the `unawaited_futures` and `discarded_futures`
lints (covered in [code quality](09-code-quality.md)) exist to catch
automatically, because it's easy to miss in review.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `Future.wait([...])` | Run futures concurrently, wait for all, results in input order |
| `Completer<T>` | Bridge a callback-based API into a `Future` |
| `.complete()` / `.completeError()` | Call exactly once per `Completer` — a second call throws |
| `eagerError: true` (default) | `await Future.wait(...)` returns as soon as the first error appears |
| `eagerError: false` | Waits for every future to settle before reporting an error |
| Missing `await` on an `async` call | Its exceptions become unhandled — the surrounding `try/catch` never sees them |
| `unawaited_futures` lint | Flags exactly this missing-`await` mistake at analysis time |

## Exercise

Write a function `Future<List<String>> fetchAll(List<Future<String> Function()> tasks)`
that runs all tasks concurrently with `Future.wait` and returns their
results. Then write a second version, `fetchAllTolerant`, that instead runs
each task individually, catches any error per-task, and returns a
`List<String>` where failed tasks are replaced with `'ERROR: $message'`
rather than aborting the whole batch. Demonstrate both with a mix of
succeeding and failing tasks (`Future.delayed` + a conditional `throw`), and
write one deliberately-broken version of `fetchAll` that forgets `await` on
one of the tasks to see the unhandled-exception crash for yourself, then fix it.
