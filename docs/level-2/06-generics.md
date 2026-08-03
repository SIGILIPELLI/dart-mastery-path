# 06 · Generics

You've been using generics since Level 1 every time you wrote `List<int>` or
`Map<String, dynamic>` — `List` and `Map` are themselves generic classes.
This module covers writing your **own** generic classes and functions: code
that works across many types while still giving you full compile-time type
checking, instead of falling back to `dynamic` and losing that safety.

## A generic class

`Box<T>` declares a type parameter `T` that's filled in when the class is
used — `Box<int>`, `Box<String>`, `Box<Score>`, all from one class
definition.

```dart
class Box<T> {
  T value;
  Box(this.value);

  void replace(T newValue) {
    value = newValue;
  }

  @override
  String toString() => 'Box($value)';
}

void main() {
  final intBox = Box<int>(42);
  final stringBox = Box<String>('hello');
  print(intBox);     // Box(42)
  print(stringBox);  // Box(hello)

  intBox.replace(100);
  print(intBox); // Box(100)

  // intBox.replace('oops'); // Error: 'oops' isn't a subtype of int
}
```

The commented-out line is the entire point: the compiler enforces that
`intBox` only ever holds `int`s, at zero runtime cost — no casting, no
`is` checks needed later.

## A generic function

Type parameters work on functions too, not just classes. Dart can usually
**infer** `T` from the arguments, so you rarely need to write it explicitly
at the call site.

```dart
T firstOrDefault<T>(List<T> items, T defaultValue) {
  return items.isEmpty ? defaultValue : items.first;
}

void main() {
  print(firstOrDefault<int>([1, 2, 3], 0));  // 1
  print(firstOrDefault<int>([], 0));         // 0
  print(firstOrDefault(['a', 'b'], 'none')); // a -- T inferred as String
}
```

## Multiple type parameters

A generic type can take more than one parameter — a `Pair<K, V>` needs both
a key type and a value type, for example.

```dart
class Pair<K, V> {
  final K key;
  final V value;
  Pair(this.key, this.value);

  @override
  String toString() => '$key -> $value';
}

void main() {
  final pair = Pair<String, int>('age', 30);
  print(pair); // age -> 30

  final pairs = [Pair('a', 1), Pair('b', 2)];
  for (final p in pairs) {
    print(p);
  }
  // a -> 1
  // b -> 2
}
```

## Bounded type parameters — `T extends SomeType`

Sometimes generic code needs to *call methods* on `T`, not just store and
return it. `T extends Comparable<T>` restricts what can fill in `T`, while
guaranteeing every valid `T` supports `compareTo` — the class can then use
it safely.

```dart
class Leaderboard<T extends Comparable<T>> {
  final List<T> _entries = [];

  void add(T entry) => _entries.add(entry);

  T get highest {
    // Safe to call compareTo -- the bound guarantees every T has it.
    return _entries.reduce((a, b) => a.compareTo(b) >= 0 ? a : b);
  }
}

class Score implements Comparable<Score> {
  final String player;
  final int points;
  Score(this.player, this.points);

  @override
  int compareTo(Score other) => points.compareTo(other.points);

  @override
  String toString() => '$player: $points';
}

void main() {
  final board = Leaderboard<Score>();
  board.add(Score('Ada', 50));
  board.add(Score('Grace', 90));
  board.add(Score('Alan', 70));
  print(board.highest); // Grace: 90
}
```

Without the `extends Comparable<T>` bound, `a.compareTo(b)` wouldn't
compile — the analyzer has no way to know an arbitrary `T` supports it.
`Leaderboard<int>` also works out of the box, since `int` already implements
`Comparable<int>`.

## The trap: generics aren't just "typed `dynamic`"

It's tempting to reach for `dynamic` instead of a type parameter when
writing reusable code — they can look similar at a glance, but they behave
completely differently. `dynamic` opts a value **out** of compile-time
checking entirely; a generic type parameter keeps full checking, just
parameterized.

```dart
class Stack<T> {
  final List<T> _items = [];

  void push(T item) => _items.add(item);

  T pop() {
    if (_items.isEmpty) throw StateError('Cannot pop from an empty stack');
    return _items.removeLast();
  }

  bool get isEmpty => _items.isEmpty;
}

void main() {
  final stack = Stack<int>();
  stack.push(1);
  stack.push(2);
  // stack.push('three'); // Error: caught at compile time, not runtime

  print(stack.pop()); // 2 -- still an int, no cast needed

  // The type parameter is preserved at runtime too, and checkable:
  print(stack is Stack<int>); // true
}
```

Had `Stack` been written with `List<dynamic>` instead of `List<T>`, pushing
a `String` onto an `int` stack would compile fine and blow up later, far
from the mistake, the first time something tried to use the popped value as
an `int`. Generics catch that at the call site instead.

| Approach | Compile-time safety | Typical use |
|---|---|---|
| `dynamic` | None — anything goes, checked at runtime if at all | Genuinely unknown/mixed JSON-like data |
| `T` (generic) | Full — enforced per instantiation (`Stack<int>` vs `Stack<String>`) | Reusable containers/algorithms over a caller-chosen type |
| `T extends Bound` | Full, plus lets you call `Bound`'s methods on `T` | Generic code that needs to compare, hash, or otherwise operate on `T` |

## Exercise

Write a generic class `Cache<K, V>` with a private `Map<K, V> _store`, a
method `V? get(K key)`, a method `void set(K key, V value)`, and a method
`V getOrCompute(K key, V Function() compute)` that returns the cached value
if present, otherwise calls `compute()`, stores the result, and returns it.
Test it with `Cache<String, int>`, using `getOrCompute` to memoize an
expensive-looking computation (print a message each time `compute` actually
runs, and confirm it only runs once per key across repeated calls).
