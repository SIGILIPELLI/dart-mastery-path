# 01 · Setup & First Program

## Install the Dart SDK

Dart programs run on the Dart SDK — the `dart` command-line tool that bundles
a compiler, a VM, package manager (`pub`), and standard library all in one.

```bash
# macOS (Homebrew)
brew tap dart-lang/dart
brew install dart

# Windows (Chocolatey)
choco install dart-sdk

# Linux — see https://dart.dev/get-dart for distro-specific instructions
```

If you plan to build Flutter apps later, installing the
[Flutter SDK](https://flutter.dev) instead also gives you a bundled Dart SDK —
but for this course, the standalone Dart SDK is all you need.

Verify the install:

```bash
dart --version
# Dart SDK version: 3.x.x (stable) ...
```

## Running a Dart file directly

Create `hello.dart`. Unlike Java, Dart does not require the filename to match
any class name — top-level functions are allowed, and every Dart program's
entry point is a top-level `main()` function.

```dart
// hello.dart
void main() {
  print('Hello, world!');
}
```

Run it directly with `dart run` (or just `dart`):

```bash
dart run hello.dart
# Hello, world!
```

There is no separate manual compile step for everyday development — `dart run`
JIT-compiles and executes in one step, which is why the edit-run loop feels
closer to a scripting language than to Java or C++.

## Compiling to a standalone executable

For production or CLI tools, compile ahead-of-time (AOT) to a fast, dependency-free
native binary:

```bash
dart compile exe hello.dart -o hello
./hello
# Hello, world!
```

This produces a self-contained executable — no Dart SDK needed on the machine
that runs it.

## Anatomy of the program

| Piece | Meaning |
|-------|---------|
| `void main()` | The program's entry point. Returns nothing (`void`). |
| `main(List<String> args)` | The entry point can optionally accept command-line arguments. |
| `print(...)` | Writes text followed by a newline to standard output. |
| `;` | Every statement ends with a semicolon. |
| `{ }` | Curly braces delimit blocks — function bodies, class bodies, loop bodies. |
| `//` | Starts a single-line comment. |

## Command-line arguments

```dart
// greet.dart
void main(List<String> args) {
  if (args.isEmpty) {
    print('Usage: dart run greet.dart <name>');
    return;
  }
  print('Hello, ${args[0]}!');
}
```

```bash
dart run greet.dart Ada
# Hello, Ada!
```

## Choosing an editor

**VS Code** with the official "Dart" extension is the most common free setup —
it gives you autocomplete, inline errors, formatting, and a debugger.
**IntelliJ IDEA** / **Android Studio** with the Dart plugin work well too,
especially if you're heading toward Flutter. You can also experiment instantly
in the browser at [DartPad](https://dartpad.dev) with zero install — handy for
trying out the snippets in this course before you have the SDK set up locally.

## Formatting your code

Dart ships an opinionated formatter, so style debates mostly don't happen:

```bash
dart format hello.dart
```

Running `dart format .` on a whole project keeps every file in a consistent
style automatically.

## Exercise

Write a program `greeter.dart` whose `main` function reads command-line
arguments and prints a personalized greeting for each name passed in (e.g.
`dart run greeter.dart Ada Grace Linus` should print three greeting lines).
If no arguments are given, print a usage message instead. Then compile it to a
standalone executable with `dart compile exe` and run the resulting binary
directly.
