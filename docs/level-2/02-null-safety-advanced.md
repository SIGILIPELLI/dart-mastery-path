# 02 · Null Safety Advanced

[Level 1](../level-1/07-null-safety-basics.md) covered the basics: `?` marks
a type nullable, `??` provides a fallback, and the compiler forces you to
handle the `null` case before using a value. This module goes deeper into
the tools and traps that show up once you're writing real, non-trivial Dart:
`late`, promotion rules, the `!` operator, and the migration gotchas that
trip people up when null safety meets existing code.

## `late` — promise now, initialize later

`late` tells the compiler "trust me, this will be assigned before anything
reads it" — letting you declare a non-nullable field without providing a
value immediately. It's essential for fields set up in a separate
initialization step (dependency injection, `initState`-style setup) where a
constructor-time value isn't available.

```dart
class Repository {
  // `late` promises "this will be set before it's read" -- and defers
  // initialization until first access, without forcing a nullable type.
  late final Config config;

  void configure(Config c) {
    config = c;
  }
}

class Config {
  String? _cachedName;

  String get name {
    // late-init-by-hand pattern: compute once, cache, reuse.
    return _cachedName ??= _expensiveLookup();
  }

  String _expensiveLookup() {
    print('doing expensive lookup...');
    return 'production';
  }
}

void main() {
  final repo = Repository();
  repo.configure(Config());
  print(repo.config.name);
  // doing expensive lookup...
  // production

  final repo2 = Repository();
  print(repo2.config.name); // never configured
}
```

Forgetting to call `configure()` before reading `config` doesn't fail at
compile time — it's a runtime crash:

```
Unhandled exception:
LateInitializationError: Field 'config' has not been initialized.
```

`late` trades a compile-time guarantee for a runtime one. Use it when you're
confident about initialization order (e.g. Flutter's `initState`), not as a
generic escape hatch to silence the analyzer.

## Null-aware operators, all together

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

String? findCityUpper(Person p) {
  // Null-aware chaining: short-circuits to null the instant any link is null.
  return p.address?.city?.toUpperCase();
}

void main() {
  final withCity = Person('Ada', Address('London'));
  final withoutAddress = Person('Grace');

  print(findCityUpper(withCity));       // LONDON
  print(findCityUpper(withoutAddress)); // null

  // Null-coalescing assignment: only assigns if currently null.
  String? nickname;
  nickname ??= 'Unknown';
  print(nickname); // Unknown

  String? maybeName;
  print(maybeName ?? 'Anonymous'); // Anonymous
}
```

| Operator | Meaning |
|---|---|
| `?.` | Call/access only if the receiver isn't null; otherwise the whole expression is `null` |
| `??` | "Or else" — use the right side if the left side is `null` |
| `??=` | Assign only if the variable is currently `null` |
| `!` | Assert non-null; throws at runtime if wrong |

## The `!` (bang) operator — a promise, not a fix

`!` tells the compiler to treat a nullable value as non-null right now. It's
sometimes unavoidable, but it's also the single most common way null-safety
bugs sneak back in — because it turns a compile-time category of bug back
into a runtime one.

```dart
int riskyLength(String? maybe) {
  return maybe!.length;
}

void main() {
  riskyLength(null);
}
// Unhandled exception:
// Null check operator used on a null value
```

Treat `!` as a note-to-self that says "I haven't proven this to the
compiler yet" — every `!` in a codebase is worth a second look during review.

## Promotion — and where it silently stops working

After a null check, Dart usually "promotes" a nullable variable to its
non-nullable type for the rest of that scope, so you don't need `!`
afterward. But promotion has real limits, and hitting one is a classic
"why won't this compile" moment for people new to the language.

```dart
class Cache {
  String? value; // mutable, non-final field

  void printLength() {
    // Promotion does NOT apply to a mutable field, even right after a null
    // check -- the analyzer can't prove nothing else (a getter, another
    // method, a callback) mutates `value` between the check and the use.
    if (value != null) {
      // print(value.length); // Error: value is still String? here
      print(value!.length); // needs the bang, or...
    }
  }

  void printLengthFixed() {
    // ...the idiomatic fix: copy to a local variable. Locals CAN be
    // promoted, because the analyzer fully controls their scope.
    final local = value;
    if (local != null) {
      print(local.length); // promoted -- no bang needed
    }
  }
}

void main() {
  final cache = Cache()..value = 'cached data';
  cache.printLength();      // 11
  cache.printLengthFixed(); // 11
}
```

**Promotion works for:** local variables, and `final` fields declared in the
same class. **Promotion does not work for:** mutable (non-`final`) fields,
getters, and fields from another library/class — because in each of those
cases something else could theoretically change the value between your
check and your use. The fix is always the same: copy the nullable value into
a local variable first, then check and use the local.

## Migration gotchas

Two things trip people up most often when adapting older, pre-null-safety
patterns or APIs to a null-safe codebase:

- **A default parameter value doesn't make a type non-nullable.**
  `void greet({String name = 'friend'})` — `name` is `String`, not `String?`,
  *inside* the function, but callers can still only omit it, not pass `null`
  explicitly, unless you write `String?` and handle it yourself.
- **JSON and other dynamic data are not automatically null-checked.**
  `json['age'] as int` compiles fine but throws a `TypeError` at runtime if
  the key is missing — casting past `dynamic` re-opens the door null safety
  otherwise keeps shut. Use `json['age'] as int?` and handle `null`
  explicitly, or validate before casting. [Module 5](05-json.md) covers this
  in depth.

## Exercise

Write a class `Profile` with a mutable nullable field `String? bio`. Add a
method `String bioPreview()` that returns the first 20 characters of `bio`
followed by `'...'` if it's longer than 20 characters, or `'No bio yet'` if
`bio` is `null` or empty — using the local-variable-promotion pattern from
this module (no `!`). Then write a top-level function `int? parseAge(Map<String,
dynamic> json)` that safely extracts an `'age'` key as an `int?`, returning
`null` if the key is missing or not an integer, without letting a bad cast
throw.
