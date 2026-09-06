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

## How It Actually Works

`dart run` and `dart compile exe` are not just two flags on the same engine —
they use fundamentally different execution strategies:

- **`dart run` (JIT path).** The Dart VM parses your source into an
  intermediate representation and starts executing it through an
  interpreter/baseline-JIT almost immediately — that's why there's no visible
  "compiling..." pause. As functions get called repeatedly (rare for a
  one-shot script, common in long-running servers), the VM's profiling counters
  flag "hot" functions and hand them to an optimizing compiler that generates
  machine code specialized for the types actually observed at runtime
  (speculative optimization). If a later call violates those assumptions —
  say a variable that was always an `int` suddenly holds a `String` — the VM
  deoptimizes back to unoptimized code rather than crashing.
- **`dart compile exe` (AOT path).** Here there is no runtime profiling step
  at all. The AOT compiler performs whole-program type-flow analysis ahead of
  time, generates native machine code for every reachable function, and links
  it with a minimal Dart runtime (a small snapshot of the core libraries plus
  a garbage collector) into one executable. This is why AOT binaries start in
  milliseconds with no warm-up and no `dart` SDK dependency — the "VM" you're
  running is really just a runtime shim, not the full JIT compiler.

The `void main()` entry point matters mechanically too: the compiled/interpreted
program's isolate (Dart's unit of concurrency — see the Level 3 isolates
lesson) starts by scheduling `main()` on its event loop. Nothing else runs
until `main` returns *and* the event loop's queues (microtasks, then
event-queue callbacks like timers) drain — which is why a synchronous
`print` inside `main` always fires before, say, a `Future.delayed` callback
scheduled earlier in the same function.

## Exercise

Write a program `greeter.dart` whose `main` function reads command-line
arguments and prints a personalized greeting for each name passed in (e.g.
`dart run greeter.dart Ada Grace Linus` should print three greeting lines).
If no arguments are given, print a usage message instead. Then compile it to a
standalone executable with `dart compile exe` and run the resulting binary
directly.
