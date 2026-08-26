# 08 · Performance & Profiling

Every prior module cared about correctness; this one is about speed and
measuring it honestly. Dart's rule of thumb is the same as everywhere else:
**measure before optimizing**, because intuition about what's slow is wrong
often enough to be dangerous. This module covers `Stopwatch`,
`dart:developer`'s `Timeline`, and one of the most common Dart-specific
performance traps.

## Measuring with `Stopwatch`

`Stopwatch` is the simplest tool: start it, run code, read the elapsed time.

```dart
void main() {
  final sw = Stopwatch()..start();
  var total = 0;
  for (int i = 0; i < 10000000; i++) {
    total += i;
  }
  sw.stop();
  print('Sum: $total, took ${sw.elapsedMilliseconds}ms');
}
// Sum: 49999995000000, took 12ms
```

A single run is noisy — background work, JIT warm-up, and GC pauses all
shift the number. For anything you're making a real decision from, run it
several times (or use the `benchmark_harness` package, which handles
warm-up iterations for you) rather than trusting one number.

## The trap: `String +=` in a loop is quadratic

`String` in Dart is immutable — every `+=` doesn't mutate the existing
string, it **allocates an entirely new one** and copies the old contents
into it. In a loop, that turns what looks like linear work into quadratic
work.

```dart
String concatWithPlus(int n) {
  String s = '';
  for (int i = 0; i < n; i++) {
    s += 'x'; // each += allocates a whole new string, O(n) per iteration
  }
  return s;
}

String concatWithBuffer(int n) {
  final buffer = StringBuffer();
  for (int i = 0; i < n; i++) {
    buffer.write('x'); // appends into a growable internal buffer
  }
  return buffer.toString();
}

void main() {
  const n = 200000;

  final sw1 = Stopwatch()..start();
  concatWithPlus(n);
  sw1.stop();
  print('String += : ${sw1.elapsedMilliseconds}ms');

  final sw2 = Stopwatch()..start();
  concatWithBuffer(n);
  sw2.stop();
  print('StringBuffer: ${sw2.elapsedMilliseconds}ms');
}
// String += : 1029ms
// StringBuffer: 2ms
```

That's a ~500x difference for 200,000 characters, and the gap only widens
as `n` grows, because `+=` is O(n) *per call* while `StringBuffer.write` is
amortized O(1) per call. Any loop building up a string — logging, report
generation, HTML/JSON assembly by hand — should reach for `StringBuffer`
(or `List<String>.join()`) instead of repeated `+=`.

## Tagging work with `dart:developer`'s `Timeline`

For anything more structured than "how long did this whole function take,"
`Timeline.startSync`/`finishSync` mark named spans that show up in DevTools'
timeline view (and in Flutter's performance overlay) — useful once you're
profiling a real app rather than a standalone benchmark.

```dart
import 'dart:developer' as dev;

int fib(int n) => n < 2 ? n : fib(n - 1) + fib(n - 2);

void main() {
  dev.Timeline.startSync('fib-30');
  final result = fib(30);
  dev.Timeline.finishSync();
  print('fib(30) = $result');
}
// fib(30) = 832040
```

Running this under `dart run --observe` and opening DevTools' CPU profiler
would show a `fib-30` span with `fib`'s recursive calls nested inside it —
the same API Flutter's own framework code uses to label frame-building work
in the DevTools timeline.

## Other common Dart performance traps

- **Rebuilding collections instead of reusing them** — `list.map(...).toList()`
  inside a hot loop allocates a new list every call; hoist it out if the
  input doesn't change per-iteration.
- **`List.contains` on a large list** — O(n) per check; switch to a `Set`
  (O(1) average) if you're doing membership checks repeatedly.
- **Synchronous heavy work on the main isolate** — see
  [isolates](05-isolates.md): a slow loop blocks *everything else*,
  including a Flutter UI's frame rendering, not just the calling function.

## Cheat sheet

| Tool/trap | What it's for |
|---|---|
| `Stopwatch()..start()` / `.elapsedMilliseconds` | Quick, ad-hoc timing |
| Multiple runs, not one | A single measurement is noisy (JIT warm-up, GC) |
| `String +=` in a loop | Quadratic — allocates and copies on every iteration |
| `StringBuffer` / `List<String>.join()` | Linear — the fix for repeated string building |
| `dart:developer` `Timeline.startSync`/`finishSync` | Named spans visible in DevTools' profiler |
| `Set` vs `List` for membership checks | O(1) average vs O(n) per `.contains()` call |
| CPU-bound work on the main isolate | Blocks everything else sharing that isolate |

## Exercise

Write two functions that both build a `List<int>` of the squares of
`0..n-1`: one using `list.add(i * i)` inside a plain `for` loop after
pre-sizing with `List.filled(n, 0, growable: false)` and indexed
assignment, and one using repeated `list.add(...)` on a `List<int>` grown
from empty. Time both for `n = 5000000` with `Stopwatch` and print the
result. Then wrap the growable version in `Timeline.startSync`/
`finishSync` under a named span, and note in a comment what you'd expect to
see if you opened this in DevTools' timeline view.
