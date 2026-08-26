# 01 · Flutter Fundamentals

Everything so far has been plain Dart running on the command line. Flutter is
a UI toolkit written in Dart: you describe what the screen should look like
as a tree of **widgets**, and Flutter takes care of turning that description
into pixels on iOS, Android, web, and desktop. This module is a Dart
developer's tour of the widget model — enough to read and write basic
Flutter code, not a full UI course.

!!! note "This module needs the Flutter SDK"
    Everything in Levels 1–2 ran with plain `dart run`. The snippets below
    need `flutter run` (a device or simulator) to actually show a screen.
    We call out where output was verified with `flutter analyze` /
    `flutter test` instead of a live run.

## Everything is a widget

A widget is an immutable description of part of the UI. Flutter has two core
kinds: `StatelessWidget` (renders from its constructor arguments and never
changes on its own) and `StatefulWidget` (paired with a `State` object that
can hold mutable data and rebuild itself).

```dart
import 'package:flutter/material.dart';

class Greeting extends StatelessWidget {
  const Greeting({super.key, required this.name});

  final String name;

  @override
  Widget build(BuildContext context) {
    return Text('Hello, $name!');
  }
}
```

`build()` is called whenever Flutter decides this widget needs to be
re-rendered — on first display, and again any time its inputs or ancestor
state change. It should be a pure function of its fields: no side effects,
no network calls, just "given this data, return this widget tree."

## StatefulWidget: separating the widget from its state

The widget itself is thrown away and rebuilt constantly; the `State` object
that backs a `StatefulWidget` survives across rebuilds until the widget is
removed from the tree entirely.

```dart
import 'package:flutter/material.dart';

class Counter extends StatefulWidget {
  const Counter({super.key});

  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;

  void _increment() {
    setState(() {
      _count++; // mutate, then tell Flutter to rebuild
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),
        ElevatedButton(onPressed: _increment, child: const Text('+1')),
      ],
    );
  }
}
```

`setState()` does two things: runs the closure (where you actually mutate
fields), then schedules `build()` to run again so the new `_count` shows up.
Mutating `_count` *without* calling `setState` compiles fine and is a
classic trap — the value changes, but the screen never updates because
Flutter was never told to rebuild.

## The trap: losing state by rebuilding the wrong node

Flutter matches old and new widget trees by **type and position** (or by
`key` when position isn't enough) to decide whether to reuse a `State`
object or throw it away and create a fresh one. Wrapping a stateful widget in
a new parent, or changing its type conditionally, can silently reset it.

```dart
// Verified with `flutter analyze` (no device needed for this check).
import 'package:flutter/material.dart';

class Toggler extends StatefulWidget {
  const Toggler({super.key});
  @override
  State<Toggler> createState() => _TogglerState();
}

class _TogglerState extends State<Toggler> {
  bool _wrapped = false;

  @override
  Widget build(BuildContext context) {
    // Each rebuild alternates whether Counter is wrapped in a Padding.
    // Flutter sees a *different tree shape* at Counter's position each
    // time, so it discards the old Counter State and creates a new one --
    // the count silently resets to 0 even though nothing "reset" it.
    final counter = const Counter();
    return Column(
      children: [
        _wrapped ? Padding(padding: const EdgeInsets.all(8), child: counter) : counter,
        ElevatedButton(
          onPressed: () => setState(() => _wrapped = !_wrapped),
          child: const Text('Toggle wrapper'),
        ),
      ],
    );
  }
}
```

The fix is a stable `key` on `Counter` so Flutter can recognize it as "the
same widget" regardless of what wraps it: `Counter(key: const ValueKey('c'))`.

## Layout without a device: widget tests

You don't need a simulator to check that a widget builds and responds to
taps — `flutter_test` renders widgets in memory and lets you assert on the
result. This is the fastest Flutter feedback loop and the one used to verify
every snippet in this module.

```dart
// test/counter_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('tapping +1 increments the count', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: Counter()));

    expect(find.text('Count: 0'), findsOneWidget);

    await tester.tap(find.text('+1'));
    await tester.pump(); // process the setState-triggered rebuild

    expect(find.text('Count: 1'), findsOneWidget);
  });
}
```

Run with `flutter test`:

```
00:01 +1: All tests passed!
```

`pump()` is the key call: Flutter doesn't rebuild synchronously inside
`tester.tap`, so forgetting `pump()` after an interaction is the single most
common reason a widget test sees stale UI and fails an assertion that looks
like it should obviously pass.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `StatelessWidget` | Renders purely from constructor fields, no internal mutable state |
| `StatefulWidget` + `State` | Widget is rebuilt often; `State` object persists across rebuilds |
| `build(BuildContext)` | Pure function: current state/config in, widget tree out |
| `setState(fn)` | Mutate state inside `fn`, then schedule a rebuild |
| `key` | Tells Flutter "this is the same logical widget" across tree shape changes |
| `flutter test` + `flutter_test` | Render widgets in memory; no simulator required |
| `tester.pump()` | Advance a frame so `setState` rebuilds are reflected before assertions |

## Exercise

Build a `StatefulWidget` called `LikeButton` that shows a heart icon and a
like count starting at 0, and increments the count by 1 each time the icon
is tapped (use an `IconButton` wrapping `Icons.favorite_border`). Write a
`flutter_test` widget test that pumps `LikeButton` inside a `MaterialApp`,
taps the icon twice, and asserts the displayed count text reads `"2"`. Then
introduce the "rebuild resets state" trap on purpose — wrap `LikeButton` in
a widget that alternates its parent type on each rebuild — and confirm with
a test that the count resets, before fixing it with a `key`.
