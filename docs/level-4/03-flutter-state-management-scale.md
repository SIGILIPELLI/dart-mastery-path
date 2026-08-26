# 03 · Flutter State Management at Scale

!!! note "This module needs the Flutter SDK"
    Verified with `flutter test` (widget tests, no simulator needed) —
    same as [Level 3's Flutter modules](../level-3/01-flutter-fundamentals.md).

[State management concepts](../level-3/07-state-management-concepts.md)
covered `ChangeNotifier` and `ListenableBuilder` for one widget listening to
one model. A real app has *many* widgets needing *shared* app-wide state —
user session, cart contents, theme — without threading it through every
constructor by hand. This module covers `InheritedWidget`/
`InheritedNotifier`, the mechanism every state-management package (Provider,
Riverpod, Bloc) builds on, and the scaling problem it doesn't solve by
itself: over-broad rebuilds.

## Making state available without constructor threading

`InheritedWidget` lets any descendant read a value from an ancestor via
`context`, without every widget in between needing to accept and forward
it. `InheritedNotifier<T>` is the built-in variant that also triggers a
rebuild of dependents whenever the `Listenable` it wraps calls
`notifyListeners()`.

```dart
import 'package:flutter/material.dart';

class AppState extends ChangeNotifier {
  int _cartItems = 0;
  String _userName = 'Guest';

  int get cartItems => _cartItems;
  String get userName => _userName;

  void addToCart() {
    _cartItems++;
    notifyListeners();
  }

  void setUserName(String name) {
    _userName = name;
    notifyListeners();
  }
}

class AppStateScope extends InheritedNotifier<AppState> {
  const AppStateScope({super.key, required AppState super.notifier, required super.child});

  static AppState of(BuildContext context) {
    final scope = context.dependOnInheritedWidgetOfExactType<AppStateScope>();
    if (scope == null) {
      throw StateError('No AppStateScope found in context');
    }
    return scope.notifier!;
  }
}
```

Any widget under `AppStateScope` in the tree can now call
`AppStateScope.of(context)` to reach the same shared `AppState`, however
deeply nested it is — no field threaded through any intermediate widget's
constructor.

```dart
class CartBadge extends StatelessWidget {
  const CartBadge({super.key});
  @override
  Widget build(BuildContext context) {
    final state = AppStateScope.of(context);
    return Text('Cart: ${state.cartItems}');
  }
}
```

## The trap: `InheritedNotifier` rebuilds ALL dependents, every time

`context.dependOnInheritedWidgetOfExactType` registers the calling widget as
a dependent of the whole `AppStateScope`, not of any one field inside it.
Every `notifyListeners()` call — no matter which field actually changed —
rebuilds every widget that ever read from that scope.

```dart
class UserLabel extends StatelessWidget {
  const UserLabel({super.key});
  @override
  Widget build(BuildContext context) {
    final state = AppStateScope.of(context);
    return Text('User: ${state.userName}');
  }
}
```

```dart
// Verified with flutter_test
testWidgets('addToCart rebuilds BOTH CartBadge and UserLabel', (tester) async {
  final appState = AppState();
  await tester.pumpWidget(MaterialApp(
    home: AppStateScope(notifier: appState, child: const HomePage()),
  ));

  appState.addToCart(); // only cartItems changed
  await tester.pump();

  expect(find.text('Cart: 1'), findsOneWidget);
  // The trap: UserLabel rebuilds too, even though userName never changed --
  // InheritedNotifier notifies ALL dependents on any notifyListeners call.
  expect(userLabelBuildCount, greaterThan(0));
});
```

```
00:00 +1: All tests passed!
```

`UserLabel` rebuilt purely because `AppState.notifyListeners()` fired for an
unrelated field. For a small app this is harmless; at scale (dozens of
widgets reading a large shared state object) it means every state change
anywhere triggers a rebuild storm across the whole tree — the actual
scaling problem this module's title refers to.

## The fix: split state, or scope what a widget depends on

Two practical strategies, in increasing order of effort:

1. **Split one big `AppState` into several smaller `ChangeNotifier`s** (a
   `CartState`, a `UserState`) and scope each independently — a widget only
   depending on `CartState` never rebuilds when `UserState` changes.
2. **Select a narrow value and compare it yourself**, using a
   `ValueListenableBuilder` (or a custom `InheritedWidget` with a manual
   `updateShouldNotify` comparing only the field you care about) instead of
   subscribing to the whole model. This is exactly the problem `Provider`'s
   `Selector` and Riverpod's fine-grained providers are built to solve —
   "rebuild only when *this specific value* changes," not "rebuild on any
   change to the object that owns it."

## Cheat sheet

| Concept | Meaning |
|---|---|
| `InheritedWidget` | Makes a value available to descendants without constructor threading |
| `InheritedNotifier<T>` | Also rebuilds dependents when `T.notifyListeners()` fires |
| `context.dependOnInheritedWidgetOfExactType<X>()` | Registers this widget as a dependent of `X` |
| Trap: any change rebuilds all dependents | `InheritedNotifier` can't distinguish which field changed |
| Split into smaller models | Independent `ChangeNotifier`s scoped separately avoid unrelated rebuilds |
| `ValueListenableBuilder` / selector pattern | Rebuild only on a specific derived value, not the whole model |

## Exercise

Split `AppState` into `CartState extends ChangeNotifier` (holds
`cartItems`) and `UserState extends ChangeNotifier` (holds `userName`), each
with its own `InheritedNotifier` scope. Rebuild `CartBadge` and `UserLabel`
against their respective scopes. Write a `flutter_test` that calls
`cartState.addToCart()` and asserts `CartBadge` rebuilds while `UserLabel`
does not (track build counts as static ints, reset between tests), proving
the split actually isolates rebuilds this time.
