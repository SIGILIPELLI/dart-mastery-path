# 07 · State Management Concepts

[Flutter fundamentals](01-flutter-fundamentals.md) used `setState` inside a
single `State` object — fine for state that only one widget cares about.
"State management" is really about one question: when data changes, how do
the (possibly many, possibly distant) widgets that display it find out?
Every Flutter state-management approach — `ChangeNotifier`, `Provider`,
Riverpod, Bloc — is a variation on the same core idea: an observable holder
of state, and something that rebuilds when it changes.

## The core idea, without any package

Strip away the Flutter-specific names and it's the Observer pattern from
[design patterns](04-design-patterns.md): a value, a list of listeners, and
a setter that notifies them.

```dart
class Store<T> {
  Store(this._value);
  T _value;
  final List<void Function(T)> _listeners = [];

  T get value => _value;

  set value(T newValue) {
    _value = newValue;
    for (final listener in List.of(_listeners)) {
      listener(newValue);
    }
  }

  void addListener(void Function(T) listener) => _listeners.add(listener);
  void removeListener(void Function(T) listener) => _listeners.remove(listener);
}

void main() {
  final cart = Store<int>(0);

  void onCartChanged(int items) => print('Cart badge: $items');
  cart.addListener(onCartChanged);

  cart.value = 1;
  cart.value = 2;

  cart.removeListener(onCartChanged);
  cart.value = 3; // no listener fires -- nothing printed for this one
  print('Final value: ${cart.value}');
}
// Cart badge: 1
// Cart badge: 2
// Final value: 3
```

Note the `List.of(_listeners)` when notifying — iterating a **copy**, not
the live list. That single line is the difference between this working and
the trap below.

## The trap: mutating the listener list while notifying it

If a listener callback adds or removes a listener (very common — a widget
unsubscribing itself, or subscribing a new one in response to a change),
iterating the live list throws.

```dart
class BadStore<T> {
  BadStore(this._value);
  T _value;
  final List<void Function(T)> _listeners = [];

  set value(T newValue) {
    _value = newValue;
    // BUG: iterating the live list while a listener can mutate it.
    for (final listener in _listeners) {
      listener(newValue);
    }
  }

  void addListener(void Function(T) listener) => _listeners.add(listener);
  void removeListener(void Function(T) listener) => _listeners.remove(listener);
}

void main() {
  final store = BadStore<int>(0);
  late void Function(int) a;
  a = (v) {
    print('A saw $v, unsubscribing');
    store.removeListener(a);
  };
  void b(int v) => print('B saw $v');

  store.addListener(a);
  store.addListener(b);

  try {
    store.value = 1;
  } catch (e) {
    print('Caught: $e');
  }
}
// A saw 1, unsubscribing
// Caught: Concurrent modification during iteration: Instance(length:1) of '_GrowableList'.
```

`B` never gets notified at all — the exception cuts the loop short after
`A` mutates the list mid-iteration. This exact bug (iterating a live
subscriber list) is why Flutter's real `ChangeNotifier` internally
snapshots its listener list before calling them, same as `List.of(...)`
above.

## Flutter's version: `ChangeNotifier` + `ListenableBuilder`

Flutter ships this pattern ready-made. `ChangeNotifier` provides
`addListener`/`removeListener`/`notifyListeners()`; `ListenableBuilder`
subscribes to a `Listenable` and rebuilds its `builder` on every
notification — no `setState` boilerplate required in the listening widget.

```dart
import 'package:flutter/material.dart';

class CartModel extends ChangeNotifier {
  int _items = 0;
  int get items => _items;

  void add() {
    _items++;
    notifyListeners();
  }
}

class CartBadge extends StatelessWidget {
  const CartBadge({super.key, required this.model});
  final CartModel model;

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: model,
      builder: (context, _) => Text('Items: ${model.items}'),
    );
  }
}
```

```dart
// test/cart_badge_test.dart -- verified with `flutter test`
testWidgets('CartBadge rebuilds when the model notifies', (tester) async {
  final model = CartModel();
  await tester.pumpWidget(MaterialApp(home: CartBadge(model: model)));

  expect(find.text('Items: 0'), findsOneWidget);

  model.add();
  await tester.pump();

  expect(find.text('Items: 1'), findsOneWidget);
});
```

```
00:00 +1: All tests passed!
```

`CartBadge` is a plain `StatelessWidget` — it holds no mutable state of its
own. `ListenableBuilder` is the piece doing the subscribing and rebuilding,
which is exactly the same shape as `addListener`/notify above, just wired
into Flutter's widget tree.

## Where the bigger packages fit in

`Provider`, Riverpod, and Bloc all solve a problem this simple example
doesn't: *how do deeply nested widgets get access to a `Store`/
`ChangeNotifier` without passing it down through every constructor?* They
use `InheritedWidget` (a Flutter mechanism for making data available to any
descendant widget without threading it through parameters) plus varying
opinions on testability, boilerplate, and compile-time safety. The core
rebuild-on-notify mechanism underneath is the same one covered here.

## Cheat sheet

| Concept | Meaning |
|---|---|
| Observable + listeners | Same Observer pattern as [design patterns](04-design-patterns.md) |
| `List.of(_listeners)` before notifying | Snapshot the list — avoids concurrent-modification when a listener mutates it |
| `ChangeNotifier` | Flutter's built-in observable base class |
| `notifyListeners()` | Tells every registered listener "rebuild now" |
| `ListenableBuilder` | Subscribes to a `Listenable`, rebuilds `builder` on notify |
| `InheritedWidget` | Makes data available to descendants without constructor threading |
| Provider / Riverpod / Bloc | Layer more structure/testability on top of the same notify-and-rebuild core |

## Exercise

Extend `CartModel` with a `remove()` method (clamped at 0, never negative)
and a computed `String get summary` that returns `"empty"` when `items ==
0` and `"$items item(s)"` otherwise. Build a widget `CartSummary` using
`ListenableBuilder` to show `summary`, plus two buttons wired to `add()` and
`remove()`. Write a `flutter_test` that pumps the widget, taps add three
times and remove once, and asserts the summary reads `"2 item(s)"`; then
taps remove three more times and asserts it reads `"empty"` (never a
negative count).
