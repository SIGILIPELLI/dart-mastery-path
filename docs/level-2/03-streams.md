# 03 · Streams

[Async basics](../level-1/08-async-basics.md) introduced `Future` for a
single value that arrives later, plus a quick taste of `Stream`. A `Stream<T>`
is the sequence version: it can emit **zero, one, or many** values over time
— a good fit for user input events, WebSocket messages, file-watching, or
any "things keep happening" scenario. This module goes deep on how to
produce, consume, transform, and safely handle errors from streams.

## Producing a stream with `async*`

`async*` marks a generator function that can `yield` values over time
instead of returning one value with `return`.

```dart
Stream<int> countUpTo(int max) async* {
  for (int i = 1; i <= max; i++) {
    await Future.delayed(const Duration(milliseconds: 100));
    yield i; // emit one value and keep going
  }
}

Future<void> main() async {
  // `await for` consumes a stream one value at a time, in order.
  await for (final n in countUpTo(3)) {
    print('Got: $n');
  }
  print('Done');
  // Got: 1
  // Got: 2
  // Got: 3
  // Done
}
```

## Two ways to consume a stream

`await for` reads naturally but pauses the enclosing `async` function until
the stream finishes. `.listen()` is the lower-level, callback-based
alternative — it doesn't block, and gives you a `StreamSubscription` you can
cancel early.

```dart
Future<void> main() async {
  final sub = countUpTo(2).listen(
    (value) => print('listen got: $value'),
    onDone: () => print('listen done'),
  );

  await Future.delayed(const Duration(milliseconds: 300));
  await sub.cancel(); // stop listening -- important to avoid leaks
}

Stream<int> countUpTo(int max) async* {
  for (int i = 1; i <= max; i++) {
    await Future.delayed(const Duration(milliseconds: 100));
    yield i;
  }
}
// listen got: 1
// listen got: 2
// listen done
```

Always keep the `StreamSubscription` and cancel it when you're done
listening (e.g. when a widget is disposed) — an uncancelled subscription
keeps its callback (and everything it closes over) alive for as long as the
stream lives.

## The trap: single-subscription vs. broadcast

By default, streams (including anything from `async*`) are
**single-subscription** — they can only be listened to **once, ever**.
Listening a second time throws.

```dart
Stream<int> countUpTo(int max) async* {
  for (int i = 1; i <= max; i++) {
    yield i;
  }
}

Future<void> main() async {
  final single = countUpTo(3);
  single.listen((v) => print('first: $v'));

  try {
    single.listen((v) => print('second: $v'));
  } catch (e) {
    print('Caught: $e');
    // Caught: Bad state: Stream has already been listened to.
  }
}
```

If you need multiple independent listeners (typical for UI events or
anything "broadcast" by nature), use a `StreamController.broadcast()`:

```dart
import 'dart:async';

Future<void> main() async {
  final controller = StreamController<int>.broadcast();

  controller.stream.listen((v) => print('listener A: $v'));
  controller.stream.listen((v) => print('listener B: $v'));

  controller.add(10);
  controller.add(20);
  await Future.delayed(const Duration(milliseconds: 50));
  await controller.close();
  // listener A: 10
  // listener B: 10
  // listener A: 20
  // listener B: 20
}
```

A broadcast stream drops events that arrive before anyone is listening, and
it doesn't buffer — every current listener gets every event from the point
they subscribed onward, not from the beginning.

| | Single-subscription | Broadcast |
|---|---|---|
| Listeners allowed | Exactly one, ever | Any number |
| Created via | `async*`, `Stream.fromIterable`\*, etc. | `StreamController.broadcast()` |
| Typical use | One-shot data pipelines, file reads | UI events, sockets, pub/sub |

\* Some built-in single-subscription-labeled streams (like
`Stream.fromIterable`) are lenient about re-listening; don't rely on that —
generator-based (`async*`) streams reliably enforce the single-listener rule,
and code should be written as if any single-subscription stream will.

## Transforming streams: `map`, `where`, and friends

Streams support the same functional-style methods as `Iterable` — `map`,
`where`, `take`, `skip` — except they operate lazily, value by value, as
each one arrives.

```dart
Future<void> main() async {
  final evensDoubled = Stream.fromIterable([1, 2, 3, 4, 5, 6])
      .where((n) => n.isEven)
      .map((n) => n * 2);

  await for (final v in evensDoubled) {
    print('transformed: $v');
  }
  // transformed: 4
  // transformed: 8
  // transformed: 12
}
```

## Error handling in streams

An error inside a stream is its own kind of event — distinct from a done
value — and `await for` and `.listen()` handle it differently.

```dart
Stream<int> riskyNumbers() async* {
  yield 1;
  yield 2;
  yield* Stream.error(Exception('something went wrong'));
  yield 3; // still runs -- an error event doesn't end the stream by itself
}

Future<void> main() async {
  // await-for: an error is thrown like a normal exception, ending the loop.
  try {
    await for (final n in riskyNumbers()) {
      print('Value: $n');
    }
  } catch (e) {
    print('Caught in await-for: $e');
  }
  // Value: 1
  // Value: 2
  // Caught in await-for: Exception: something went wrong

  // listen(): onError fires, but -- unless you pass cancelOnError: true --
  // the subscription is NOT automatically cancelled, so later events
  // (including "3" here) can still arrive.
  riskyNumbers().listen(
    (n) => print('listen value: $n'),
    onError: (e) => print('listen onError: $e'),
    onDone: () => print('listen onDone'),
  );
  // listen value: 1
  // listen value: 2
  // listen onError: Exception: something went wrong
  // listen value: 3
  // listen onDone
}
```

This asymmetry is a genuine gotcha: code that works correctly with
`await for` (loop stops on error) can behave completely differently once
rewritten with `.listen()` (keeps going unless you opt in with
`cancelOnError: true` or cancel manually inside `onError`).

## Cheat sheet

| Concept | Meaning |
|---|---|
| `async*` / `yield` | Define a stream that emits values over time |
| `yield*` | Emit every value from another stream (delegate) |
| `await for` | Consume a stream synchronously-looking; stops on error |
| `.listen(onData, onError, onDone)` | Consume via callbacks; errors don't auto-stop it |
| `StreamController.broadcast()` | Create a stream multiple listeners can subscribe to |
| `.map()` / `.where()` | Lazily transform stream values as they arrive |
| `cancelOnError: true` | Make `.listen()` behave like `await for` on error |

## Exercise

Write a function `Stream<int> countdown(int from)` that yields values from
`from` down to `1`, one every 300ms, then throws an error instead of a final
`0`. Consume it twice: once with `await for` inside a `try`/`catch`, and once
with `.listen()` passing `cancelOnError: true` so the two behave the same
way. Then build a `StreamController<String>.broadcast()` with two listeners
that print incoming messages with different prefixes (e.g. `"[A] $msg"` and
`"[B] $msg"`), add three messages to it, and close it.
