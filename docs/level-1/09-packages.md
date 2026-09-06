# 09 · Packages (pub, pubspec.yaml)

## What is `pub`?

`pub` is Dart's package manager (bundled with the SDK), and
[pub.dev](https://pub.dev) is the public package registry — Dart's equivalent
of npm or PyPI. Every Dart project is described by a `pubspec.yaml` file at
its root.

## Creating a new project

```bash
dart create my_app
cd my_app
```

This scaffolds a small console application, including a `pubspec.yaml`, a
`bin/my_app.dart` entry point, and a `test/` folder pre-wired for the `test`
package.

## Anatomy of `pubspec.yaml`

```yaml
name: my_app
description: A sample command-line application.
version: 1.0.0
publish_to: 'none'   # prevents accidental publishing to pub.dev

environment:
  sdk: ^3.4.0

dependencies:
  http: ^1.2.0
  args: ^2.5.0

dev_dependencies:
  test: ^1.25.0
  lints: ^4.0.0
```

| Section | Meaning |
|---------|---------|
| `name` | The package's own name (lowercase, underscores) |
| `version` | Follows semantic versioning: `major.minor.patch` |
| `environment.sdk` | Which Dart SDK versions this package supports |
| `dependencies` | Packages your production code needs |
| `dev_dependencies` | Packages only needed for testing/tooling, not shipped |

## Adding dependencies

```bash
dart pub add http
dart pub add --dev test
```

This updates `pubspec.yaml` automatically and fetches the package. Running it
manually after hand-editing the file works too:

```bash
dart pub get
```

`pub get` resolves every dependency's version (writing the exact resolved
versions into `pubspec.lock` so builds are reproducible) and downloads
packages into a local cache.

## Using a package

```dart
// bin/my_app.dart
import 'package:http/http.dart' as http;

Future<void> main() async {
  var response = await http.get(Uri.parse('https://example.com'));
  print('Status code: ${response.statusCode}');
}
```

```bash
dart run bin/my_app.dart
# Status code: 200
```

## Version constraints

```yaml
dependencies:
  http: ^1.2.0       # caret syntax -- allows 1.2.0 <= version < 2.0.0
  args: '>=2.4.0 <3.0.0'   # explicit range, equivalent meaning
  path: any                # any version (avoid in real projects)
```

The caret (`^`) constraint is the most common — it follows semantic
versioning, allowing patch and minor updates but never an accidental breaking
major-version bump.

## Organizing your own project into a library

Real packages split reusable code into `lib/`, keeping `bin/` for thin
executable entry points.

```
my_app/
├── pubspec.yaml
├── bin/
│   └── my_app.dart        # entry point -- imports from lib/
├── lib/
│   └── src/
│       └── calculator.dart
├── test/
│   └── calculator_test.dart
```

```dart
// lib/src/calculator.dart
int add(int a, int b) => a + b;
```

```dart
// bin/my_app.dart
import 'package:my_app/src/calculator.dart';

void main() {
  print(add(2, 3));   // 5
}
```

Importing `package:my_app/src/calculator.dart` (rather than a relative path)
works because `pub` registers your own project as an importable package named
after `pubspec.yaml`'s `name` field.

## Useful `pub` commands

| Command | Purpose |
|---------|---------|
| `dart create <name>` | Scaffold a new project |
| `dart pub get` | Install/resolve dependencies from `pubspec.yaml` |
| `dart pub add <package>` | Add and install a new dependency |
| `dart pub add --dev <package>` | Add a dev-only dependency (testing, linting) |
| `dart pub upgrade` | Upgrade dependencies within their version constraints |
| `dart pub outdated` | Show which dependencies have newer versions available |
| `dart pub deps` | Print the resolved dependency tree |

## How It Actually Works

Running `dart pub get` doesn't just download files — it runs a real
**version-solving** algorithm (PubGrub, the same family of SAT-based solver
used by other modern package managers) that must simultaneously satisfy
every version constraint declared across your `pubspec.yaml` *and* every
transitive dependency's own `pubspec.yaml`. The output is written to
`pubspec.lock`, which pins the exact resolved version of every package —
that's why committing `pubspec.lock` (for applications, not for published
packages) matters: without it, two machines running `pub get` against the
same loose constraints (`^1.2.0`) could resolve to different concrete
versions if a new compatible release was published in between.

`^1.2.0` isn't a convention Dart merely documents — it's parsed into a
concrete version range object (`>=1.2.0 <2.0.0`) by the pub tool itself,
following semantic versioning's contract that a major version bump may
break compatibility. The solver treats this range as a hard constraint
during solving, not a soft preference.

Once resolution finishes, `pub` doesn't copy the resolved packages into your
project — it writes a `.dart_tool/package_config.json` file mapping each
package name to an on-disk path (in the global pub cache, typically
`~/.pub-cache`), and the Dart compiler/VM consults that file to resolve
`import 'package:foo/foo.dart'` at compile time. This is why deleting
`.dart_tool` and re-running `pub get` is a safe way to "reset" a project's
dependency resolution without touching the shared, deduplicated pub cache
that every project on your machine draws from.

## Exercise

Run `dart create demo_cli` to scaffold a new console project. Add the `args`
package as a dependency with `dart pub add args`, then edit
`bin/demo_cli.dart` to use `ArgParser` from that package to accept a
`--name` flag, printing a greeting for the given name (or "World" if not
provided). Run it with `dart run bin/demo_cli.dart --name=Ada`.
