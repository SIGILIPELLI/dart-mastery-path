# 07 · Null Safety Basics

## What "sound null safety" means

Since Dart 2.12, every type is **non-nullable by default**. A variable
declared `String name` can never hold `null` — the compiler guarantees it, not
just at your call sites but everywhere in the whole program. To allow `null`,
you opt in explicitly with a `?` suffix on the type.

```dart
void main() {
  String name = 'Ada';
  // name = null;   // compile error: null can't be assigned to non-nullable String

  String? maybeName = 'Ada';
  maybeName = null;   // fine -- String? explicitly allows null
  print(maybeName);   // null
}
```

This eliminates most `NullPointerException`-style crashes at compile time
rather than runtime — if your code compiles, the compiler has already proven
every non-nullable variable is never null when used.

## The `?` nullable type suffix

```dart
void main() {
  int? age;                 // defaults to null, since no initializer given
  print(age);                // null

  age = 30;
  print(age + 1);            // 31 -- Dart knows age is non-null here (type promotion)
}
```

## The `!` null-assertion operator

`!` tells the compiler "trust me, this is not null right now" — it throws at
runtime if you're wrong, so use it sparingly and only when you're certain.

```dart
void main() {
  String? maybeText = fetchText();
  // print(maybeText.length);   // compile error -- maybeText might be null

  print(maybeText!.length);     // 5 -- asserts non-null, then accesses .length
}

String? fetchText() => 'hello';
```

```dart
void main() {
  String? risky;
  print(risky!.length);   // throws at runtime: "Null check operator used on a null value"
}
```

## The `??` and `??=` null-coalescing operators

```dart
void main() {
  String? nickname;
  String display = nickname ?? 'Anonymous';   // use right-hand side if left is null
  print(display);   // Anonymous

  nickname ??= 'Ada';    // assign only if nickname is currently null
  print(nickname);      // Ada

  nickname ??= 'Grace';  // no-op, nickname is already non-null
  print(nickname);      // Ada
}
```

## The `?.` null-aware access operator

```dart
class Address {
  String? city;
  Address(this.city);
}

class Person {
  String name;
  Address? address;
  Person(this.name, [this.address]);
}

void main() {
  var p1 = Person('Ada', Address('London'));
  var p2 = Person('Grace');   // no address

  print(p1.address?.city);   // London
  print(p2.address?.city);   // null -- short-circuits safely, no crash

  // Chain multiple ?. together
  print(p2.address?.city?.toUpperCase());   // null
}
```

## Null-aware operators for collections

```dart
void main() {
  List<int>? numbers;

  print(numbers?.length);        // null -- safe access on a possibly-null list
  print(numbers?.isEmpty ?? true);   // true

  Map<String, int>? scores;
  print(scores?['Ada'] ?? 0);    // 0
}
```

## Type promotion

Dart's analyzer tracks null checks and "promotes" a nullable variable to its
non-nullable type within the scope where it's known to be safe.

```dart
void printLength(String? text) {
  if (text != null) {
    // Inside this block, `text` is promoted to String (non-nullable)
    print(text.length);
  } else {
    print('No text provided');
  }
}

void main() {
  printLength('hello');   // 5
  printLength(null);      // No text provided
}
```

## Late variables

`late` promises the compiler a non-nullable variable will be initialized
before it's first read — useful when initialization can't happen at
declaration time (e.g. depends on constructor logic run afterward).

```dart
class Config {
  late String environment;   // will be set before use, trust me

  void load() {
    environment = 'production';
  }
}

void main() {
  var config = Config();
  config.load();
  print(config.environment);   // production

  // var badConfig = Config();
  // print(badConfig.environment);   // runtime error: LateInitializationError
}
```

## Cheat sheet

| Operator | Meaning |
|----------|---------|
| `String?` | Nullable type — may hold `null` |
| `!` | Assert non-null now; throws at runtime if wrong |
| `??` | Fallback value if the left side is `null` |
| `??=` | Assign only if currently `null` |
| `?.` | Access a member only if the receiver isn't `null` |
| `late` | Defer non-null initialization until before first use |

## Exercise

Write a function `String describeUser(String? name, int? age)` that returns
`"Name: <name>, Age: <age>"` using `??` to substitute `"Unknown"` for a null
name and `0` for a null age. Then model a `Team` class with a nullable
`Person? captain` field, and write a function that prints the captain's name
in upper case using `?.` chaining, falling back to `"No captain assigned"` if
there is none.
