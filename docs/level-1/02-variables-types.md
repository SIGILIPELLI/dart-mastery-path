# 02 · Variables & Types

## Declaring variables

Dart is statically typed but supports type inference with `var` — the type is
locked in at compile time based on the initializer, it's just not written out.

```dart
void main() {
  var name = 'Ada';       // inferred as String
  var age = 36;           // inferred as int
  var height = 1.68;      // inferred as double
  var isStudent = false;  // inferred as bool

  print('$name is $age years old.');
  // Ada is 36 years old.
}
```

You can also write the type explicitly, which is common in public APIs and
class fields for readability:

```dart
String name = 'Ada';
int age = 36;
double height = 1.68;
bool isStudent = false;
```

## Core built-in types

| Type | Example | Notes |
|------|---------|-------|
| `int` | `42`, `-7` | Arbitrary-precision on native platforms |
| `double` | `3.14`, `2.0` | 64-bit floating point |
| `num` | `int` or `double` | Common supertype of both |
| `String` | `'hello'`, `"hello"` | Single or double quotes both work |
| `bool` | `true`, `false` | No implicit truthy/falsy conversions |
| `List<T>` | `[1, 2, 3]` | Ordered, growable array |
| `Map<K, V>` | `{'a': 1}` | Key-value pairs |
| `Set<T>` | `{1, 2, 3}` | Unordered, unique elements |

## `final` and `const`

Both prevent reassignment, but `const` is stronger — it means "compile-time
constant," fixed forever, while `final` just means "set once, at runtime."

```dart
void main() {
  final currentYear = 2026;      // set once; could depend on runtime data
  const pi = 3.14159;            // must be knowable at compile time

  // currentYear = 2027;   // compile error: can't reassign a final variable
  // const now = DateTime.now(); // compile error: not a compile-time constant
}
```

Prefer `const` when the value truly never changes across runs (like a
mathematical constant); use `final` for values computed once per run (like
reading a config value or a `DateTime.now()` timestamp).

## Numbers and operators

```dart
void main() {
  int a = 17;
  int b = 5;

  print(a + b);   // 22
  print(a - b);   // 12
  print(a * b);   // 85
  print(a / b);   // 3.4        -- always returns a double
  print(a ~/ b);  // 3          -- truncating (integer) division
  print(a % b);   // 2          -- remainder

  print(a > b && b > 0);  // true
  print(a.isEven);        // false
  print(b.isOdd);         // true
}
```

Notice `/` always produces a `double`, even for two `int`s — use `~/` when you
want integer division.

## Strings and interpolation

```dart
void main() {
  String first = 'Ada';
  String last = 'Lovelace';

  // String interpolation -- ${expression} for expressions, $var for simple names
  String full = '$first $last';
  print(full);                        // Ada Lovelace
  print('${first.length} letters');   // 3 letters

  // Multi-line strings with triple quotes
  String bio = '''
$full
Mathematician & writer
''';
  print(bio);

  // Raw strings ignore escape sequences
  print(r'No \n escaping happens here');
  // No \n escaping happens here
}
```

## Type conversion

```dart
void main() {
  String numText = '42';
  int parsed = int.parse(numText);         // String -> int
  double parsedD = double.parse('3.14');   // String -> double

  int n = 42;
  String backToText = n.toString();        // int -> String

  print(parsed + 1);       // 43
  print(parsedD * 2);      // 6.28
  print('$backToText!');   // 42!

  // int.tryParse returns null instead of throwing on bad input
  int? maybe = int.tryParse('not a number');
  print(maybe);   // null
}
```

## `dynamic` vs `var` vs `Object`

```dart
void main() {
  var a = 10;        // type fixed at compile time as int -- a = 'text' is an error
  dynamic b = 10;     // type checking deferred to runtime -- b = 'text' is allowed
  Object c = 10;      // statically typed as Object -- c = 'text' is allowed (String is an Object)

  b = 'now a string';        // fine, no compile error
  print(b);                  // now a string

  // c.length;   // compile error -- Object doesn't have .length, need a cast first
}
```

In everyday Dart code, prefer `var` (or an explicit type) — `dynamic` opts out
of Dart's static type checking and should be reserved for genuinely dynamic
data like decoded JSON.

## Cheat sheet

| Keyword | Meaning |
|---------|---------|
| `var` | Type inferred once at declaration, then fixed |
| `final` | Assignable once, at runtime |
| `const` | Assignable once, at compile time |
| `dynamic` | Type checked at runtime only |
| `Object` / `Object?` | Statically typed as "any object" (nullable variant allows `null`) |

## How It Actually Works

`var` and `dynamic` look similar on the page but produce completely different
compiled code:

- With `var`, type inference happens once, at compile time, from the
  initializer expression. The compiler then bakes that concrete type into
  every subsequent use — a `var name = 'Ada'` becomes, as far as the compiler
  and the AOT/JIT backends are concerned, indistinguishable from `String name
  = 'Ada'`. Field and method lookups on `name` resolve to fixed offsets/vtable
  slots at compile time, which is what makes them fast and lets the analyzer
  catch `name = 42` before you ever run the program.
- With `dynamic`, the compiler emits no static type information for that
  variable at all. Every member access (`b.length`, `b + 1`) becomes a
  dynamic dispatch — at runtime, the VM looks up the object's actual runtime
  class, checks whether it implements the requested method/getter, and calls
  it, or throws a `NoSuchMethodError` if it doesn't. This is strictly slower
  than a statically-resolved call and defeats the analyzer's ability to catch
  mistakes ahead of time — the cost isn't just stylistic.

`final` vs `const` is also a real compile-time/runtime split, not just a
convention: a `const` value is folded directly into the compiled constant
pool (for collections, this means `const [1, 2, 3]` is a single canonicalized,
immutable object reused everywhere it's referenced — two `const [1,2,3]`
literals are `identical()`), while `final` allocates a normal object at
runtime whose reference simply can't be reassigned afterward. That's why
`const now = DateTime.now()` is rejected — the compiler cannot compute
"now" while compiling your source.

Integer division producing a `double` for `/` but truncating for `~/` is
rooted in Dart's number model: on native (JIT/AOT) targets, `int` is a true
64-bit integer type, so `~/` performs actual integer truncating division in
hardware, while `/` always promotes both operands to `double` and uses
floating-point division — two different CPU instructions, not just two
different print formats.

## Exercise

Write a program that declares your name (`String`), birth year (`int`), and
height in meters (`double`) using `var`. Compute your age given a current year
constant, then print a sentence using string interpolation that includes your
name, computed age, and height. Also demonstrate `int.tryParse` by attempting
to parse both a valid and an invalid numeric string and printing both results.
