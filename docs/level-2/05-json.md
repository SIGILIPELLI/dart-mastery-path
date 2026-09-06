# 05 · Working with JSON

JSON is the lingua franca of web APIs, config files, and inter-service
communication — the [Weather CLI project](10-project-weather-cli.md) later
in this level talks to a real API entirely through it. Dart's built-in
`dart:convert` library handles the raw encode/decode; the pattern of turning
that raw, untyped data into real Dart objects (and back) is something you
write yourself, and it's worth understanding well since [Advanced Null
Safety](02-null-safety-advanced.md) already warned that this is exactly
where null-safety guarantees can quietly break down.

## Decoding and encoding with `dart:convert`

```dart
import 'dart:convert';

void main() {
  const jsonString = '''
  {
    "name": "Ada Lovelace",
    "age": 36,
    "isActive": true,
    "skills": ["math", "programming"]
  }
  ''';

  // jsonDecode gives back plain dynamic Dart objects: Map, List, String,
  // num, bool, or null -- there's no compile-time type safety here yet.
  final decoded = jsonDecode(jsonString) as Map<String, dynamic>;

  print(decoded['name']);             // Ada Lovelace
  print(decoded['age']);              // 36
  print(decoded['skills']);           // [math, programming]
  print(decoded['age'].runtimeType);  // int

  // Encoding goes the other direction: Dart objects -> JSON text.
  final data = {
    'title': 'Dart Basics',
    'pages': 42,
    'tags': ['dart', 'programming'],
  };
  print(jsonEncode(data));
  // {"title":"Dart Basics","pages":42,"tags":["dart","programming"]}
}
```

`jsonDecode` returns `dynamic` all the way down — a `Map<String, dynamic>`
whose values might themselves be `Map`, `List`, `String`, `num`, `bool`, or
`null`. That flexibility is exactly why you don't want to pass it around
your app directly — you want a typed model.

## Building typed models with `fromJson`/`toJson`

The standard pattern: give each class a `factory` constructor that reads a
decoded `Map` and a `toJson()` method that produces one back. Nested objects
decode recursively — each level calls its own `fromJson`.

```dart
import 'dart:convert';

class Address {
  final String city;
  final String country;
  Address({required this.city, required this.country});

  factory Address.fromJson(Map<String, dynamic> json) {
    return Address(
      city: json['city'] as String,
      country: json['country'] as String,
    );
  }

  Map<String, dynamic> toJson() => {'city': city, 'country': country};
}

class Person {
  final String name;
  final int age;
  final Address address;

  Person({required this.name, required this.age, required this.address});

  factory Person.fromJson(Map<String, dynamic> json) {
    return Person(
      name: json['name'] as String,
      age: json['age'] as int,
      // Nested objects need their own fromJson call -- decoding is recursive.
      address: Address.fromJson(json['address'] as Map<String, dynamic>),
    );
  }

  Map<String, dynamic> toJson() => {
        'name': name,
        'age': age,
        'address': address.toJson(),
      };

  @override
  String toString() => '$name ($age) from ${address.city}, ${address.country}';
}

void main() {
  const raw = '''
  {
    "name": "Grace Hopper",
    "age": 85,
    "address": {"city": "New York", "country": "USA"}
  }
  ''';

  final person = Person.fromJson(jsonDecode(raw) as Map<String, dynamic>);
  print(person); // Grace Hopper (85) from New York, USA

  print(jsonEncode(person.toJson()));
  // {"name":"Grace Hopper","age":85,"address":{"city":"New York","country":"USA"}}
}
```

## The trap: casts fail at runtime, not compile time

`json['age'] as int` type-checks fine — the compiler has no way to know
what's actually in that `Map<String, dynamic>` until the program runs. A
missing key, a `null` value, or a server sending a `String` where you expect
an `int` all produce the same kind of failure: a `TypeError`, thrown the
instant the bad cast executes, potentially deep inside a function far from
where the JSON was received.

```dart
const badJson = '{"name": "No Age Here"}';

void main() {
  Person.fromJson(jsonDecode(badJson) as Map<String, dynamic>);
}
// Unhandled exception:
// type 'Null' is not a subtype of type 'int' in type cast
```

This is the concrete downside [Module 2](02-null-safety-advanced.md) warned
about: casting past `dynamic` re-opens exactly the class of bug null safety
otherwise prevents. Two defenses, often used together:

1. **Cast to a nullable type and supply a default**, so a missing/bad field
   degrades gracefully instead of crashing:

```dart
factory Book.fromJson(Map<String, dynamic> json) {
  return Book(
    title: json['title'] as String? ?? 'Untitled',
    year: json['year'] as int? ?? 0,
  );
}
```

2. **Wrap the whole decode in a `try`/`catch`** at the boundary where JSON
   enters your app (an HTTP response handler, a file read), so one bad
   payload can't crash something far away — see [Error
   Handling](08-error-handling.md) for the general pattern.

## Decoding a JSON array into a `List` of models

```dart
import 'dart:convert';

class Book {
  final String title;
  final int year;
  Book({required this.title, required this.year});

  factory Book.fromJson(Map<String, dynamic> json) {
    return Book(
      title: json['title'] as String? ?? 'Untitled',
      year: json['year'] as int? ?? 0,
    );
  }

  @override
  String toString() => '$title ($year)';
}

void main() {
  const raw = '''
  [
    {"title": "Dart in Action", "year": 2020},
    {"title": "Effective Dart"}
  ]
  ''';

  final decoded = jsonDecode(raw) as List<dynamic>;

  final books = decoded
      .map((item) => Book.fromJson(item as Map<String, dynamic>))
      .toList();

  for (final book in books) {
    print(book);
  }
  // Dart in Action (2020)
  // Effective Dart (0)     <- missing "year" defaulted safely, no crash
}
```

## Cheat sheet

| Task | Code |
|---|---|
| Parse a JSON string | `jsonDecode(text)` → `dynamic` (usually `Map`/`List`) |
| Produce a JSON string | `jsonEncode(value)` |
| Safe field access | `json['key'] as String?` + `?? default` |
| Nested object | Call the nested type's own `fromJson` |
| Array of objects | `(decoded as List).map((e) => T.fromJson(e)).toList()` |
| Round-trip a model | `T.fromJson(jsonDecode(text))` then `jsonEncode(model.toJson())` |

## How It Actually Works

`jsonDecode` performs a single-pass parse of the input string into a tree of
plain Dart objects (`Map<String, dynamic>`, `List<dynamic>`, `String`,
`num`, `bool`, `Null`) with no knowledge of your model classes at all — the
"typed model" layer (`fromJson`/`toJson`) is entirely hand-written or
generated code sitting *on top of* that untyped tree, not part of the
`dart:convert` decoding step itself. This separation is exactly why a cast
like `json['age'] as int` can fail at runtime instead of compile time: the
compiler has no way to know what shape the decoded `dynamic` map actually
has until the cast executes and the runtime checks the actual object's
class against `int`.

Because `num` (not `int` or `double` specifically) is what JSON numbers
decode to when there's no decimal point ambiguity, a JSON value like `42`
decodes to a Dart `int`, while `42.0` decodes to a `double` — and casting
`json['x'] as int` on a value that happens to arrive as `42.0` from an API
that changed its serialization throws a runtime `TypeError`, not a silent
truncation. This is a common real-world JSON-model bug precisely because
Dart's static type system can't see through the `dynamic` boundary to catch
it ahead of time.

`toJson()` produces exactly the same kind of untyped tree (`Map<String,
dynamic>`) that `jsonDecode` produces — `jsonEncode` then walks that tree
recursively, calling `.toString()`/formatting primitives and recursing into
nested maps/lists, which is why any object placed into a `toJson()` map that
isn't itself a `Map`, `List`, `String`, `num`, `bool`, or `null` (or doesn't
implement its own `toJson()` that `jsonEncode` knows to call) throws a
`JsonUnsupportedObjectError` at the point of encoding, not at the point you
constructed the object.

## Exercise

Model a small `Product` class with `name` (`String`), `price` (`double`),
and `inStock` (`bool`, defaulting to `true` if the JSON omits it). Write
`fromJson`/`toJson`, then decode this JSON array into a `List<Product>`:
`'[{"name": "Widget", "price": 9.99}, {"name": "Gadget", "price": 19.99,
"inStock": false}]'`. Print each product, then filter the list down to only
`inStock` products using `.where()`, and print the total price of just
those with `.fold()`.
