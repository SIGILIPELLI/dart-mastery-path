# 09 · Code Generation

[JSON](../level-2/05-json.md) covered writing `fromJson`/`toJson` by hand.
That's fine for one class; it gets tedious and error-prone (typo a field
name once and you have a silent bug) across dozens of models. Dart's
answer is **code generation**: annotate a class, run a build tool, and let
generated Dart code do the repetitive part — kept in sync with your class
automatically every time you regenerate it.

```yaml
# pubspec.yaml
dependencies:
  json_annotation: ^4.9.0
dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.8.0
```

## Annotating a class

`@JsonSerializable()` marks a class for generation; `part 'user.g.dart';`
tells Dart that a file generated alongside this one supplies the rest of
the class's implementation.

```dart
// lib/user.dart
import 'package:json_annotation/json_annotation.dart';

part 'user.g.dart';

@JsonSerializable()
class User {
  User({required this.name, required this.age});

  final String name;
  final int age;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}
```

`_$UserFromJson` and `_$UserToJson` don't exist anywhere you wrote — they're
what the generator is going to produce.

## Running the generator

```bash
dart run build_runner build
```

This scans the project for `@JsonSerializable()` classes and writes a
matching `user.g.dart` next to each annotated file:

```dart
// lib/user.g.dart -- GENERATED, do not edit by hand
part of 'user.dart';

User _$UserFromJson(Map<String, dynamic> json) => User(
      name: json['name'] as String,
      age: (json['age'] as num).toInt(),
    );

Map<String, dynamic> _$UserToJson(User instance) => <String, dynamic>{
      'name': instance.name,
      'age': instance.age,
    };
```

With that file in place, `User` behaves exactly like a hand-written JSON
model:

```dart
import 'dart:convert';

void main() {
  final user = User(name: 'Ada', age: 30);
  print(jsonEncode(user.toJson()));

  final decoded = User.fromJson(jsonDecode('{"name":"Grace","age":85}'));
  print('${decoded.name} is ${decoded.age}');
}
// {"name":"Ada","age":30}
// Grace is 85
```

Notice `(json['age'] as num).toInt()` rather than a plain `as int` cast —
the generator hedges against JSON numbers arriving as `double` (which is
common from JavaScript-originated APIs), something hand-written parsing
code frequently forgets to do.

## The trap: forgetting the `part` declaration

Every generated-code class needs its `part 'xxx.g.dart';` line pointing at
the file the generator will produce. Leave it out (or typo the filename)
and the generated functions genuinely don't exist as far as the analyzer is
concerned — even though `user.g.dart` might already exist on disk from a
previous run.

```dart
// lib/user.dart -- 'part' line accidentally removed
import 'package:json_annotation/json_annotation.dart';

@JsonSerializable()
class User {
  User({required this.name, required this.age});
  final String name;
  final int age;
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}
```

```
dart analyze lib/user.dart
  error - The method '_$UserFromJson' isn't defined for the type 'User'.
  error - The method '_$UserToJson' isn't defined for the type 'User'.
```

The fix is just restoring the `part` line — but the error message ("method
isn't defined") gives no hint that a missing `part` is the cause, which is
what makes this trap take longer to spot than it should the first time.

## Regenerating after model changes, and watch mode

Any time you add, remove, or rename a field, the `.g.dart` file is now
stale and must be regenerated — it does **not** happen automatically on
save. `--delete-conflicting-outputs` clears previously generated files that
would otherwise conflict (useful right after a big rename); `watch` reruns
the builder automatically on every file save during active development:

```bash
dart run build_runner build --delete-conflicting-outputs
dart run build_runner watch   # regenerate continuously while you edit
```

## Cheat sheet

| Concept | Meaning |
|---|---|
| `@JsonSerializable()` | Marks a class for `fromJson`/`toJson` generation |
| `part 'x.g.dart';` | Required — links the class to its generated implementation file |
| `dart run build_runner build` | Runs all configured generators once |
| `dart run build_runner watch` | Regenerates automatically on every save |
| `--delete-conflicting-outputs` | Clears stale generated files that would otherwise conflict |
| `.g.dart` files | Generated — never hand-edit; regenerate instead |
| `(json['x'] as num).toInt()` | Generator's defensive cast for numeric fields from JSON |

## Exercise

Create an `@JsonSerializable()` class `Product` with fields `name` (String),
`price` (double), and `inStock` (bool, defaulting to `true` when absent from
JSON using `@JsonKey(defaultValue: true)`). Run `build_runner build` to
generate its `.g.dart` file, then write a `main()` that decodes a JSON
string missing the `inStock` key and prints the resulting `Product`'s
`inStock` value (should be `true`), then encodes a second `Product` back to
a JSON string and prints it.
