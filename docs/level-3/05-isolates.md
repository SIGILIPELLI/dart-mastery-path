# 05 · Isolates

[Async basics](../level-1/08-async-basics.md) explained that Dart's event
loop runs `Future`/`async` code on a **single thread** — great for I/O,
useless for CPU-heavy work, which would block that one thread and freeze
everything else (including a Flutter UI). Isolates are Dart's answer:
independent workers with their own memory and event loop, communicating only
by passing messages.

## Spawning an isolate with a `SendPort`

`Isolate.spawn` starts a new isolate running a top-level or static function,
handing it a `SendPort` to talk back to the caller.

```dart
import 'dart:isolate';

void _worker(SendPort sendPort) {
  int sum = 0;
  for (int i = 1; i <= 1000000; i++) {
    sum += i;
  }
  sendPort.send(sum);
}

Future<void> main() async {
  final receivePort = ReceivePort();
  await Isolate.spawn(_worker, receivePort.sendPort);

  final result = await receivePort.first;
  print('Sum from isolate: $result');
}
// Sum from isolate: 500000500000
```

`receivePort.first` grabs the first message and is convenient for a
one-shot result; a long-lived worker would `listen()` on the port instead
and needs to explicitly close it (`receivePort.close()`) to avoid keeping
the program alive forever waiting for more messages.

## The simpler API: `Isolate.run`

For "run this function on another isolate and give me the result," `Isolate
.run` (added in Dart 2.19) hides the port plumbing entirely — spawn, run,
collect the result, and shut the isolate down, all in one `await`.

```dart
import 'dart:isolate';

int _fib(int n) => n < 2 ? n : _fib(n - 1) + _fib(n - 2);

Future<void> main() async {
  final sw = Stopwatch()..start();
  final result = await Isolate.run(() => _fib(32));
  print('fib(32) = $result (took ${sw.elapsedMilliseconds}ms wall, off the main isolate)');
}
// fib(32) = 2178309 (took 36ms wall, off the main isolate)
```

Reach for `Isolate.run` by default — it covers the "offload this expensive
computation" case that motivates isolates in the first place, without the
manual `ReceivePort` bookkeeping.

## The trap: no shared memory, not even globals

The single biggest mental shift coming from thread-based concurrency in
other languages: Dart isolates share **nothing** by default, not even
top-level/static variables. Each isolate gets its own copy of global state
at spawn time.

```dart
import 'dart:isolate';

int _counter = 0;

void _bumpCounter() {
  _counter++; // mutates the SPAWNED isolate's own copy, not the caller's
  print('inside isolate, counter is: $_counter');
}

Future<void> main() async {
  print('global counter before: $_counter');
  await Isolate.run(_bumpCounter);
  print('global counter after Isolate.run: $_counter'); // unchanged!
}
// global counter before: 0
// inside isolate, counter is: 1
// global counter after Isolate.run: 0
```

`_counter` in the spawned isolate is a completely separate piece of memory
from `_counter` in `main`'s isolate — incrementing one has zero effect on
the other. This is by design (it's what makes isolates safe without locks),
but it means "just use a global to share state across isolates" silently
does nothing useful. The only way data crosses an isolate boundary is
through messages sent over a port (or the return value of `Isolate.run`).

## What can and can't be sent between isolates

Messages are deep-copied, not shared by reference — and not everything can
be copied. Primitives, `String`, `List`/`Map`/`Set` of sendable types, and a
few special types (`SendPort`, `TransferableTypedData`) work. Instances of
arbitrary custom classes, open `Socket`/`File` handles, and closures that
capture non-sendable state do not survive the trip.

```dart
class NotSendable {
  NotSendable(this.value);
  final int value;
}
```

Trying to `send()` a plain custom object (without it being a "sendable"
type Dart recognizes) throws an `Invalid argument(s)` error at the send
call — a common surprise when refactoring code that used to pass Dart
objects around freely on a single isolate into isolate-based code.

## Cheat sheet

| Concept | Meaning |
|---|---|
| Isolate | Independent worker: own memory, own event loop, no shared state |
| `Isolate.spawn(fn, sendPort)` | Manual: spawn a worker, communicate via ports |
| `Isolate.run(fn)` | Simple: spawn, run, return result, auto-shutdown |
| `ReceivePort` / `SendPort` | The only channel between isolates — messages are copied |
| Global/static variables | NOT shared — each isolate gets its own independent copy |
| Sendable data | Primitives, String, collections of sendable types, `SendPort` |
| Not sendable | Arbitrary custom objects, open file/socket handles, most closures |

## Exercise

Write a function `Future<List<int>> primesUpTo(int n)` that runs on a
separate isolate via `Isolate.run` and returns all primes below `n` using a
simple trial-division sieve. Time how long `primesUpTo(200000)` takes with a
`Stopwatch`, then run the same sieve logic directly on the main isolate
(no `Isolate.run`) while printing a counter to standard output in a loop
alongside it with `Timer.periodic` — observe that the main-isolate version
blocks the periodic timer from firing until it finishes, while the
isolate-based version doesn't.
