# 08 · Async Basics

## Why async matters

Dart is single-threaded by default, but it's not blocking — I/O-bound work
(network requests, file reads, timers) runs asynchronously via an event loop,
so your program can keep doing other things while it waits. The `Future` type
represents a value that will be available *later*.

## `Future<T>`

A `Future<T>` represents a computation that will eventually produce a value of
type `T` (or fail with an error).

```dart
Future<String> fetchGreeting() {
  return Future.delayed(const Duration(seconds: 1), () => 'Hello, world!');
}

void main() {
  print('Fetching...');
  fetchGreeting().then((greeting) {
    print(greeting);
  });
  print('This prints before the greeting!');
}
// Fetching...
// This prints before the greeting!
// Hello, world!          <- appears ~1 second later
```

Notice the *order* of prints: `.then()` schedules a callback for later, so
`main` keeps running past it immediately — this is the essence of
non-blocking async code.

## `async` and `await`

`await` pauses execution *within an `async` function* until a `Future`
completes, without blocking the rest of the program. It reads like
synchronous code while still being fully non-blocking.

```dart
Future<String> fetchGreeting() {
  return Future.delayed(const Duration(seconds: 1), () => 'Hello, world!');
}

Future<void> main() async {
  print('Fetching...');
  String greeting = await fetchGreeting();   // pauses here until the Future resolves
  print(greeting);
  print('Done!');
}
// Fetching...
// Hello, world!    <- ~1 second later
// Done!
```

Any function marked `async` automatically returns a `Future` (wrapping
whatever type you `return`), even if you never explicitly write `Future<...>`.

## Error handling with `try`/`catch`

```dart
Future<int> riskyDivision(int a, int b) async {
  await Future.delayed(const Duration(milliseconds: 100));
  if (b == 0) throw ArgumentError('Cannot divide by zero');
  return a ~/ b;
}

Future<void> main() async {
  try {
    int result = await riskyDivision(10, 2);
    print('Result: $result');   // Result: 5

    int bad = await riskyDivision(10, 0);
    print('Unreachable: $bad');
  } catch (e) {
    print('Caught error: $e');
    // Caught error: Invalid argument(s): Cannot divide by zero
  }
}
```

## Running several futures concurrently with `Future.wait`

Awaiting one `Future` at a time runs them sequentially. To run independent
async operations concurrently, use `Future.wait`.

```dart
Future<String> fetchUser() async {
  await Future.delayed(const Duration(seconds: 1));
  return 'Ada';
}

Future<int> fetchScore() async {
  await Future.delayed(const Duration(seconds: 1));
  return 95;
}

Future<void> main() async {
  var stopwatch = Stopwatch()..start();

  // Both futures run concurrently -- total wait is ~1s, not ~2s
  var results = await Future.wait([fetchUser(), fetchScore()]);
  print(results);   // [Ada, 95]

  print('Took ${stopwatch.elapsedMilliseconds}ms');   // ~1000ms, not ~2000ms
}
```

## `Timer` for delayed or repeating work

```dart
import 'dart:async';

void main() {
  print('Start');

  Timer(const Duration(seconds: 1), () {
    print('One second later');
  });

  int ticks = 0;
  var repeater = Timer.periodic(const Duration(milliseconds: 500), (timer) {
    ticks++;
    print('Tick $ticks');
    if (ticks == 3) timer.cancel();
  });
}
```

## A tiny intro to `Stream`

A `Stream<T>` is like a `Future<T>` that can produce *many* values over time
instead of just one — covered in depth in Level 2. Here's a first taste:

```dart
Stream<int> countUpTo(int max) async* {
  for (int i = 1; i <= max; i++) {
    await Future.delayed(const Duration(milliseconds: 200));
    yield i;   // emit each value as it becomes ready
  }
}

Future<void> main() async {
  await for (var number in countUpTo(3)) {
    print(number);
  }
  // 1
  // 2
  // 3
}
```

## Cheat sheet

| Concept | Meaning |
|---------|---------|
| `Future<T>` | A value of type `T` that will be available later (or fail) |
| `async` | Marks a function as asynchronous; it returns a `Future` |
| `await` | Pauses inside an `async` function until a `Future` resolves |
| `.then(callback)` | Runs `callback` once the `Future` completes (non-`await` style) |
| `Future.wait([...])` | Runs multiple futures concurrently, waits for all to finish |
| `try`/`catch` | Standard way to handle errors from an `await`ed `Future` |
| `Stream<T>` | Like a `Future`, but can emit many values over time |

## Exercise

Write an `async` function `fetchTemperature(String city)` that simulates a
network call with `Future.delayed` (random 200-800ms) and returns a random
double temperature. Call it for three different cities concurrently with
`Future.wait`, print all three results, and print how long the whole
operation took using a `Stopwatch` — confirm it's close to the *slowest*
individual call, not the sum of all three.
