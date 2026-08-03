# 07 · Extension Methods

Sometimes you want to add a method to a type you don't control — a built-in
type like `String` or `int`, or a class from a package you can't edit. In
most languages you'd have to wrap it or write a free-floating helper
function. Dart's `extension` lets you add methods and getters to *any* type,
including built-ins, as if you'd written them into the original class.

## Defining an extension

```dart
extension StringCasing on String {
  String get capitalized {
    if (isEmpty) return this;
    return this[0].toUpperCase() + substring(1);
  }

  bool get isPalindrome {
    final cleaned = toLowerCase().replaceAll(RegExp(r'[^a-z0-9]'), '');
    return cleaned == cleaned.split('').reversed.join();
  }
}

void main() {
  print('hello'.capitalized);
  // Hello

  print('Was it a car or a cat I saw'.isPalindrome);
  // true

  print('hello world'.isPalindrome);
  // false
}
```

Once `StringCasing` is imported, every `String` in that file gets
`.capitalized` and `.isPalindrome` — call syntax is indistinguishable from a
method that was always part of `String`. Inside the extension body, `this`
refers to the `String` instance the method was called on, exactly like
inside a normal class method.

## Why extensions instead of a free function?

Compare `text.capitalized` to `capitalize(text)`. The extension version
reads left-to-right in the order you think about the operation, chains
naturally with other methods (`text.trim().capitalized.toUpperCase()`), and
shows up in autocomplete the moment you type `text.` — a free function
requires knowing it exists and importing it correctly, and doesn't chain as
cleanly.

## Extensions work on your own classes too

Extensions aren't just for retrofitting built-ins — they're equally useful
on your own classes, as a way to keep a core model lean and put
derived/convenience behavior (formatting, display logic, math operators)
somewhere separate. Extensions can add operators, too.

```dart
class Money {
  final int cents;
  Money(this.cents);

  @override
  String toString() => '\$${(cents / 100).toStringAsFixed(2)}';
}

// Keep display-only helpers out of the core model...
extension MoneyFormatting on Money {
  String get compact {
    if (cents >= 100000) return '\$${(cents / 100000).toStringAsFixed(1)}k';
    return toString();
  }
}

// ...and math operators can live in their own extension too.
extension MoneyMath on Money {
  Money operator +(Money other) => Money(cents + other.cents);
}

void main() {
  final price = Money(150000);
  print(price);          // $1500.00
  print(price.compact);  // $1.5k

  final total = Money(500) + Money(750);
  print(total); // $12.50
}
```

This separation is genuinely useful in larger codebases: a `Money` class
shared across a whole app might live in a core package, while
`MoneyFormatting` (which might pull in locale-specific rules) lives only in
the UI layer that actually needs it.

## Extensions can be generic too

```dart
extension ListStats<T extends num> on List<T> {
  T get sum => isEmpty ? (0 as T) : reduce((a, b) => (a + b) as T);

  double get average => isEmpty ? 0 : sum / length;
}

void main() {
  print([1, 2, 3, 4].sum);      // 10
  print([1, 2, 3, 4].average);  // 2.5
  print(<int>[].average);       // 0.0
}
```

`extension ... <T extends num> on List<T>` restricts this extension to
lists of numbers specifically — `<String>['a', 'b'].sum` simply wouldn't
compile, because `List<String>` isn't a `List<num>`.

## The trap: colliding extension names

Two different extensions can add a member with the *same* name to the same
type — perfectly legal on their own, until both happen to be imported into
the same file. Then calling that member directly becomes ambiguous, and
Dart refuses to guess which one you meant.

```dart
extension LoudA on String {
  String shout() => '${toUpperCase()}!!!';
}

extension LoudB on String {
  String shout() => '${toUpperCase()}...?!';
}

void main() {
  // print('hi'.shout());
  // Error: A member named 'shout' is defined in 'extension LoudA on String'
  // and 'extension LoudB on String', and neither is more specific.

  // The fix: disambiguate with an explicit extension override.
  print(LoudA('hi').shout());  // HI!!!
  print(LoudB('hi').shout());  // HI...?!
}
```

`ExtensionName(value).member()` — an *extension override* — is the escape
hatch: it tells the compiler exactly which extension's implementation to
use, bypassing the ambiguity. This is a real risk once a project pulls in
several packages that each add convenience extensions on common types like
`String`, `DateTime`, or `List` — naming your own extensions specifically
(`StringCasing`, not something generic) makes collisions easier to spot and
resolve.

## Extensions don't add real storage

An extension can only add computed getters/methods — it cannot add new
instance *fields*, because it isn't actually part of the class; it's syntax
sugar over a static function that takes the receiver as a hidden argument.
Anything an extension "adds" has to be derived from members the type
already has.

| Can an extension... | |
|---|---|
| Add a method/getter/setter | Yes |
| Add an operator (e.g. `+`) | Yes |
| Add a new instance field | No — no extra storage exists |
| Override an existing member | No — extensions are picked only when no real member matches |
| Be generic (`extension E<T> on List<T>`) | Yes |

## Exercise

Write an extension `DurationFormatting on Duration` with a getter
`readable` that returns a human-friendly string like `"1h 5m 30s"` (omit any
unit that's zero — e.g. a duration of exactly 90 seconds should print `"1m
30s"`, not `"0h 1m 30s"`). Test it against a few `Duration` values built with
`Duration(hours: ..., minutes: ..., seconds: ...)`, including one under a
minute and one over an hour.
