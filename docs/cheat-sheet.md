---
title: "Dart Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Dart cheat sheet: 29 quick-reference tables from every level of the Dart Mastery Path, each linked to its full lesson."
---

# Dart Cheat Sheet

Quick-reference tables from **29 lessons** of the free Dart Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

| Keyword | Meaning |
|---------|---------|
| `var` | Type inferred once at declaration, then fixed |
| `final` | Assignable once, at runtime |
| `const` | Assignable once, at compile time |
| `dynamic` | Type checked at runtime only |
| `Object` / `Object?` | Statically typed as "any object" (nullable variant allows `null`) |

### [03 · Control Flow](level-1/03-control-flow.md)

| Construct | Use when |
|-----------|----------|
| `if`/`else if`/`else` | Branching on arbitrary boolean conditions |
| `switch` statement | Branching on discrete values with side effects per case |
| `switch` expression | Producing a value from discrete cases, exhaustively checked |
| `for` | Known number of iterations, or iterating a collection |
| `while` | Unknown number of iterations, condition checked before body |
| `do-while` | Same, but body always runs at least once |
| `break` / `continue` | Exit a loop early / skip to the next iteration |
| `?:` | Inline conditional expression |
| `??` / `??=` | Fallback for `null`, or assign-if-null |

### [04 · Functions](level-1/04-functions.md)

| Syntax | Meaning |
|--------|---------|
| `int add(int a, int b) { ... }` | Standard function with a block body |
| `int square(int x) => x * x;` | Arrow syntax for single-expression bodies |
| `[String greeting = 'Hi']` | Optional positional parameter with default |
| `{required String name}` | Required named parameter |
| `{int age = 18}` | Optional named parameter with default |
| `int Function(int, int)` | Type of a function taking two ints, returning an int |

### [05 · Collections (List, Map, Set)](level-1/05-collections.md)

| Type | Literal | Ordered? | Duplicates? | Access |
|------|---------|----------|-------------|--------|
| `List<T>` | `[1, 2, 3]` | Yes | Yes | By index: `list[0]` |
| `Map<K, V>` | `{'a': 1}` | Insertion order | Keys unique | By key: `map['a']` |
| `Set<T>` | `{1, 2, 3}` | Insertion order | No | By value: `set.contains(x)` |

### [06 · Classes &amp; Objects Basics](level-1/06-classes-objects.md)

| Feature | Syntax |
|---------|--------|
| Constructor shorthand | `Person(this.name, this.age);` |
| Named constructor | `Person.guest() : name = 'Guest', age = 0;` |
| Getter | `double get area => width * height;` |
| Setter | `set scaleWidth(double f) { width *= f; }` |
| Private member | Prefix with `_`, e.g. `_balance` |
| Static member | `static const pi = 3.14159;` |

### [07 · Null Safety Basics](level-1/07-null-safety-basics.md)

| Operator | Meaning |
|----------|---------|
| `String?` | Nullable type — may hold `null` |
| `!` | Assert non-null now; throws at runtime if wrong |
| `??` | Fallback value if the left side is `null` |
| `??=` | Assign only if currently `null` |
| `?.` | Access a member only if the receiver isn't `null` |
| `late` | Defer non-null initialization until before first use |

### [08 · Async Basics](level-1/08-async-basics.md)

| Concept | Meaning |
|---------|---------|
| `Future<T>` | A value of type `T` that will be available later (or fail) |
| `async` | Marks a function as asynchronous; it returns a `Future` |
| `await` | Pauses inside an `async` function until a `Future` resolves |
| `.then(callback)` | Runs `callback` once the `Future` completes (non-`await` style) |
| `Future.wait([...])` | Runs multiple futures concurrently, waits for all to finish |
| `try`/`catch` | Standard way to handle errors from an `await`ed `Future` |
| `Stream<T>` | Like a `Future`, but can emit many values over time |

## Level 2 · Intermediate

### [03 · Streams](level-2/03-streams.md)

| Concept | Meaning |
|---|---|
| `async*` / `yield` | Define a stream that emits values over time |
| `yield*` | Emit every value from another stream (delegate) |
| `await for` | Consume a stream synchronously-looking; stops on error |
| `.listen(onData, onError, onDone)` | Consume via callbacks; errors don't auto-stop it |
| `StreamController.broadcast()` | Create a stream multiple listeners can subscribe to |
| `.map()` / `.where()` | Lazily transform stream values as they arrive |
| `cancelOnError: true` | Make `.listen()` behave like `await for` on error |

### [05 · Working with JSON](level-2/05-json.md)

| Task | Code |
|---|---|
| Parse a JSON string | `jsonDecode(text)` → `dynamic` (usually `Map`/`List`) |
| Produce a JSON string | `jsonEncode(value)` |
| Safe field access | `json['key'] as String?` + `?? default` |
| Nested object | Call the nested type's own `fromJson` |
| Array of objects | `(decoded as List).map((e) => T.fromJson(e)).toList()` |
| Round-trip a model | `T.fromJson(jsonDecode(text))` then `jsonEncode(model.toJson())` |

### [08 · Error Handling](level-2/08-error-handling.md)

| Tool | Use |
|---|---|
| `implements Exception` | Mark a custom type as an expected, recoverable failure |
| `on SomeType catch (e)` | Catch only a specific throwable type |
| `catch (e)` (bare) | Catch anything, including `Error`s — put it last |
| `finally` | Runs regardless of success/failure/rethrow |
| `rethrow` | Re-throw the current exception, preserving its original stack trace |
| `await` inside `try`/`catch` | Catches errors from the awaited `Future` normally |
| Un-awaited `Future` | Its error bypasses any surrounding `try`/`catch` |

### [09 · Collections Deep Dive](level-2/09-collections-deep-dive.md)

| Need | Reach for |
|---|---|
| Keep only matching elements | `.where((e) => condition)` |
| Transform each element | `.map((e) => newValue)` |
| Reduce to one value, safe on empty | `.fold(initial, (acc, e) => ...)` |
| Reduce to one value from the elements themselves | `.reduce((a, b) => ...)` |
| Flatten a list of lists | `.expand((e) => e.subList)` |
| Sort in place | `list.sort((a, b) => a.compareTo(b))` |
| Group by a key | `Map.putIfAbsent` inside a loop |
| Force a lazy `Iterable` to compute now | `.toList()` |

## Level 3 · Advanced

### [01 · Flutter Fundamentals](level-3/01-flutter-fundamentals.md)

| Concept | Meaning |
|---|---|
| `StatelessWidget` | Renders purely from constructor fields, no internal mutable state |
| `StatefulWidget` + `State` | Widget is rebuilt often; `State` object persists across rebuilds |
| `build(BuildContext)` | Pure function: current state/config in, widget tree out |
| `setState(fn)` | Mutate state inside `fn`, then schedule a rebuild |
| `key` | Tells Flutter "this is the same logical widget" across tree shape changes |
| `flutter test` + `flutter_test` | Render widgets in memory; no simulator required |
| `tester.pump()` | Advance a frame so `setState` rebuilds are reflected before assertions |

### [02 · Building APIs with `shelf`](level-3/02-building-apis-shelf.md)

| Concept | Meaning |
|---|---|
| `Handler` | `FutureOr<Response> Function(Request)` — the whole server model |
| `shelf_io.serve(handler, host, port)` | Bind a handler to a real socket |
| `Router()..get(path, fn)` | Route by method + path, with `<param>` captures |
| Route params | Always arrive as `String`; parse/validate yourself |
| `Pipeline().addMiddleware(...).addHandler(...)` | Compose cross-cutting behavior around a handler |
| `jsonEncode` on non-`String` map keys | Throws at runtime — stringify keys first |
| `logRequests()` | Built-in middleware that prints one line per request |

### [03 · Databases](level-3/03-databases.md)

| Concept | Meaning |
|---|---|
| `sqlite3.openInMemory()` | Throwaway DB for the process lifetime — good for tests |
| `sqlite3.open('file.db')` | Persistent DB backed by a file |
| `db.execute(sql)` | Run raw SQL, no parameter binding — never interpolate input into it |
| `db.prepare(sql)` / `.execute([...])` | Bind `?` placeholders safely — use for any dynamic value |
| `db.select(sql, [params])` | Run a query and get back rows you can index like a map |
| `.dispose()` | Required on `Database` and `PreparedStatement` — native resources, no GC finalizer |
| `SqliteException` | Thrown for constraint violations and SQL errors — catch it specifically |

### [04 · Design Patterns in Dart](level-3/04-design-patterns.md)

| Pattern | Problem it solves | Dart mechanism |
|---|---|---|
| Singleton | Exactly one shared instance | Private constructor + `factory` returning a cached field |
| Factory | Pick concrete type at runtime, hide it from callers | `factory` constructor with a `switch`/lookup |
| Observer | Broadcast state changes to many listeners | List of interfaces, loop-and-notify on mutation |
| Strategy | Swap an algorithm without changing the caller | Interface held as a mutable field |
| `identical(a, b)` | True object-identity check | Ignores any `==` override |

### [05 · Isolates](level-3/05-isolates.md)

| Concept | Meaning |
|---|---|
| Isolate | Independent worker: own memory, own event loop, no shared state |
| `Isolate.spawn(fn, sendPort)` | Manual: spawn a worker, communicate via ports |
| `Isolate.run(fn)` | Simple: spawn, run, return result, auto-shutdown |
| `ReceivePort` / `SendPort` | The only channel between isolates — messages are copied |
| Global/static variables | NOT shared — each isolate gets its own independent copy |
| Sendable data | Primitives, String, collections of sendable types, `SendPort` |
| Not sendable | Arbitrary custom objects, open file/socket handles, most closures |

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

| Concept | Meaning |
|---|---|
| `group(name, () {...})` | Namespace and organize related tests |
| `setUp(() {...})` | Runs before every test in the enclosing group — fresh fixtures |
| `tearDown(() {...})` | Runs after every test — release resources |
| Fake implementing an interface | Test double with real (simplified) behavior, no framework needed |
| `completion(matcher)` | Assert on the value a `Future` completes with |
| `throwsA(matcher)` | Assert an exception is thrown — needs `() => ...`, not a called value |
| `isA<T>()` | Matcher: value is an instance of `T` |

### [07 · State Management Concepts](level-3/07-state-management-concepts.md)

| Concept | Meaning |
|---|---|
| Observable + listeners | Same Observer pattern as [design patterns](level-3/04-design-patterns.md) |
| `List.of(_listeners)` before notifying | Snapshot the list — avoids concurrent-modification when a listener mutates it |
| `ChangeNotifier` | Flutter's built-in observable base class |
| `notifyListeners()` | Tells every registered listener "rebuild now" |
| `ListenableBuilder` | Subscribes to a `Listenable`, rebuilds `builder` on notify |
| `InheritedWidget` | Makes data available to descendants without constructor threading |
| Provider / Riverpod / Bloc | Layer more structure/testability on top of the same notify-and-rebuild core |

### [08 · Performance &amp; Profiling](level-3/08-performance-profiling.md)

| Tool/trap | What it's for |
|---|---|
| `Stopwatch()..start()` / `.elapsedMilliseconds` | Quick, ad-hoc timing |
| Multiple runs, not one | A single measurement is noisy (JIT warm-up, GC) |
| `String +=` in a loop | Quadratic — allocates and copies on every iteration |
| `StringBuffer` / `List<String>.join()` | Linear — the fix for repeated string building |
| `dart:developer` `Timeline.startSync`/`finishSync` | Named spans visible in DevTools' profiler |
| `Set` vs `List` for membership checks | O(1) average vs O(n) per `.contains()` call |
| CPU-bound work on the main isolate | Blocks everything else sharing that isolate |

### [09 · Code Generation](level-3/09-code-generation.md)

| Concept | Meaning |
|---|---|
| `@JsonSerializable()` | Marks a class for `fromJson`/`toJson` generation |
| `part 'x.g.dart';` | Required — links the class to its generated implementation file |
| `dart run build_runner build` | Runs all configured generators once |
| `dart run build_runner watch` | Regenerates automatically on every save |
| `--delete-conflicting-outputs` | Clears stale generated files that would otherwise conflict |
| `.g.dart` files | Generated — never hand-edit; regenerate instead |
| `(json['x'] as num).toInt()` | Generator's defensive cast for numeric fields from JSON |

## Level 4 · Master

### [01 · Advanced Async Patterns](level-4/01-advanced-async-patterns.md)

| Concept | Meaning |
|---|---|
| `Future.wait([...])` | Run futures concurrently, wait for all, results in input order |
| `Completer<T>` | Bridge a callback-based API into a `Future` |
| `.complete()` / `.completeError()` | Call exactly once per `Completer` — a second call throws |
| `eagerError: true` (default) | `await Future.wait(...)` returns as soon as the first error appears |
| `eagerError: false` | Waits for every future to settle before reporting an error |
| Missing `await` on an `async` call | Its exceptions become unhandled — the surrounding `try/catch` never sees them |
| `unawaited_futures` lint | Flags exactly this missing-`await` mistake at analysis time |

### [02 · Production Dart Services](level-4/02-production-dart-services.md)

| Concept | Meaning |
|---|---|
| `Config.fromEnv(Map<String,String>)` | Take env as a parameter, not `Platform.environment` directly — testable |
| JSON-per-line logging | Machine-parseable; include a UTC timestamp on every entry |
| `ProcessSignal.sigterm.watch()` | Catch shutdown signals to close cleanly instead of being killed |
| `server.close(force: false)` | Stop accepting new connections, let in-flight ones finish |
| `sigterm` on Windows | Not supported — only `sigint` works there |
| `dart run` vs `dart compile exe` | Signal delivery can differ — test against the real deployed artifact |
| `/health` endpoint | Minimal liveness/readiness check for orchestrators |

### [03 · Flutter State Management at Scale](level-4/03-flutter-state-management-scale.md)

| Concept | Meaning |
|---|---|
| `InheritedWidget` | Makes a value available to descendants without constructor threading |
| `InheritedNotifier<T>` | Also rebuilds dependents when `T.notifyListeners()` fires |
| `context.dependOnInheritedWidgetOfExactType<X>()` | Registers this widget as a dependent of `X` |
| Trap: any change rebuilds all dependents | `InheritedNotifier` can't distinguish which field changed |
| Split into smaller models | Independent `ChangeNotifier`s scoped separately avoid unrelated rebuilds |
| `ValueListenableBuilder` / selector pattern | Rebuild only on a specific derived value, not the whole model |

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

| Concept | Meaning |
|---|---|
| `sha256` alone for passwords | Fast + deterministic — wrong properties for password storage |
| Salt (`Random.secure()` bytes) | Makes identical passwords hash differently |
| bcrypt / scrypt / Argon2 | Deliberately slow — what real password hashing should use (not `sha256`) |
| `Random()` | Predictable — fine for games, unsafe for tokens/salts |
| `Random.secure()` | Cryptographically secure source — use for anything security-sensitive |
| Path traversal (`../`) | User input joined onto a path can escape the intended directory |
| `Uri.file(path).normalizePath()` | Collapses `..`/`.` before validating the path is still in-bounds |

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

| Concept | Meaning |
|---|---|
| `tags: ['slow']` on `test()` | Group tests to include/exclude by category |
| `dart_test.yaml` `tags:` block | Declares valid tags, avoids "undeclared tag" warnings |
| `dart test --exclude-tags X` | Skip tagged tests for a fast local loop |
| `dart test --coverage=DIR` | Record which lines executed |
| `format_coverage --lcov` | Convert to LCOV for CI coverage dashboards |
| Line coverage ≠ branch coverage | A "covered" line can still hide an untested conditional path |
| `dart analyze --fatal-infos` in CI | Fail the build on any analyzer issue, not just errors |

### [06 · Deployment with Docker](level-4/06-deployment-docker.md)

| Concept | Meaning |
|---|---|
| `dart compile exe` | Produces a standalone native binary, no SDK needed to run it |
| `InternetAddress.anyIPv4` | Bind to all interfaces — required to be reachable from outside a container |
| Multi-stage Dockerfile | Build with the full SDK image, ship only the compiled binary |
| `FROM scratch` + Dart's `/runtime/` | Minimal final image — tens of MB instead of a full SDK |
| `COPY pubspec.* ./` before `COPY . .` | Caches `pub get` separately so code edits don't force a re-resolve |
| `ENV PORT` + reading it in the app | Standard way to make the container's port configurable at run time |

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

| Technique | When to reach for it |
|---|---|
| `Isolate.run` per chunk | CPU-bound work large enough that isolate overhead is worth it |
| Measure actual speedup | Isolate overhead can make fine-grained chunking net slower |
| `BEGIN`/`COMMIT` around many writes | Batches disk-sync cost across many operations instead of paying it per-row |
| Implicit per-statement transactions | The default without an explicit `BEGIN` — one sync per write |
| Bounded batch size | Balances throughput against how much work one failure can roll back |
| Benchmark on real storage | In-memory vs. on-disk DBs show very different batching payoffs |

### [08 · Package Publishing (pub.dev)](level-4/08-package-publishing.md)

| File/step | Purpose |
|---|---|
| `pubspec.yaml` `description` | Shown on pub.dev listing/search — write a real sentence |
| `README.md` | Rendered as the pub.dev package page's main content |
| `CHANGELOG.md` | One entry per published version — consumers read this before upgrading |
| `LICENSE` (exact name) | Required for pub.dev's license detection and trust score |
| `dart pub publish --dry-run` | Runs every publish check without uploading |
| Doc comments (`///`) | Rendered as the generated API reference on pub.dev |
| Semver (`MAJOR.MINOR.PATCH`) | `PATCH` = fix, `MINOR` = compatible addition, `MAJOR` = breaking change |

### [09 · Code Quality (`dart analyze` / lints)](level-4/09-code-quality.md)

| Concept | Meaning |
|---|---|
| `analysis_options.yaml` | Project-level analyzer/lint configuration |
| `package:lints/core.yaml` | Minimal, correctness-focused preset |
| `package:lints/recommended.yaml` | Broader preset, adds style rules |
| `dart analyze` | Runs the configured checks, reports issues |
| `unawaited_futures` | Flags a discarded `Future` inside an `async` function — catches the missing-`await` crash |
| Lint scoped to `async` contexts only | A non-`async` caller doing the same thing isn't flagged — review those by eye |
| `dart fix --apply` | Auto-applies mechanical fixes for lints that have one |
| `// ignore: rule_name` | Silence a lint for one line, with a reason in a comment |

---

Go deeper with the full [Dart Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
