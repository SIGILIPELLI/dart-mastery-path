---
title: "Dart Interview Questions and Answers (178 Questions)"
description: "178 Dart interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Dart Mastery Path."
---

# Dart Interview Questions and Answers

**178 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Dart Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What does the `dart` command bundle?

**Answer:** A compiler, VM, package manager (pub) and tooling

One SDK for everything.

**Q2.** What does `dart run bin/main.dart` do?

**Answer:** Runs the program on the Dart VM (JIT)

dart compile makes native executables.

**Q3.** What is the entry point of a Dart program?

**Answer:** The top-level main() function

main can take List&lt;String&gt; args.

**Q4.** What does `dart create` do?

**Answer:** Scaffolds a new project with pubspec.yaml and sample code

Templates: console, package, server-shelf.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```dart
void main(){ var x = 5; var y = 2.5; print(x * y); print((x * y).runtimeType); print(7 ~/ 2); }
```

**Answer:** 12.5
double
3

~/ is integer division. Verified by running it with the Dart SDK.

**Q6.** What does this print?

```dart
void main(){ final a = [1,2]; a.add(3); const b = [1,2]; print(a); print(b.length); }
```

**Answer:** [1, 2, 3]
2

final prevents reassigning the variable, not mutating the list. Verified by running it with the Dart SDK.

**Q7.** What's the difference between final and const?

**Answer:** final is set once at runtime; const is a compile-time constant

Prefer const for constant values.

**Q8.** What does `var name = 'Ada';` infer?

**Answer:** String, fixed at compile time

var isn't dynamic.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```dart
void main(){ var s=0; for (var i=1;i<=10;i++){ if(i.isEven) continue; if(i>7) break; s+=i; } print(s); }
```

**Answer:** 16

1+3+5+7. Verified by running it with the Dart SDK.

**Q10.** What does this print?

```dart
void main(){ var n=3; var r = switch(n){ 1 => 'one', 2 || 3 => 'few', _ => 'many' }; print(r); }
```

**Answer:** few

Switch expressions (Dart 3). Verified by running it with the Dart SDK.

**Q11.** What does this print?

```dart
void main(){ for (final c in 'abc'.split('')) { print(c.toUpperCase()); } }
```

**Answer:** A
B
C

for-in iterates items. Verified by running it with the Dart SDK.

**Q12.** Is there implicit truthiness in Dart conditions?

**Answer:** No: conditions must be bool

Write explicit comparisons.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```dart
String greet(String name, {String greeting='Hello'}) => '$greeting, $name!';
void main(){ print(greet('Ada')); print(greet('Bo', greeting:'Hi')); }
```

**Answer:** Hello, Ada!
Hi, Bo!

Named optional parameters. Verified by running it with the Dart SDK.

**Q14.** What does this print?

```dart
int Function(int) adder(int n) => (x) => x + n;
void main(){ var add5 = adder(5); print(add5(10)); }
```

**Answer:** 15

Closures capture n. Verified by running it with the Dart SDK.

**Q15.** What does `=>` mean in a function?

**Answer:** Shorthand for a single expression body with an implicit return

Equivalent to { return expr; }.

**Q16.** What are positional optional parameters?

**Answer:** Parameters in [brackets] that may be omitted

e.g. void f(int a, [int b = 0]).

### [05 · Collections (List, Map, Set)](level-1/05-collections.md)

**Q17.** What does this print?

```dart
void main(){ var m = {'a':1}; m['b']=2; m.update('a',(v)=>v+5); print(m); print(m.containsKey('c')); }
```

**Answer:** {a: 6, b: 2}
false

Map literal printing. Verified by running it with the Dart SDK.

**Q18.** What does this print?

```dart
void main(){ var s = {1,2,3}; s.add(2); s.add(4); print(s.length); print(s.contains(4)); }
```

**Answer:** 4
true

Sets ignore duplicates. Verified by running it with the Dart SDK.

**Q19.** What does this print?

```dart
void main(){ var l=[3,1,2]; l.sort(); l.add(9); print(l); print(l.reversed.toList()); }
```

**Answer:** [1, 2, 3, 9]
[9, 3, 2, 1]

sort is in place. Verified by running it with the Dart SDK.

**Q20.** Which collection stores unique values?

**Answer:** Set

Use for membership tests.

### [06 · Classes &amp; Objects Basics](level-1/06-classes-objects.md)

**Q21.** What does this print?

```dart
class P{ final String name; int age; P(this.name, this.age); String toString()=>'$name($age)'; }
void main(){ var p=P('Ada',36); p.age++; print(p); }
```

**Answer:** Ada(37)

Initializing formals set fields. Verified by running it with the Dart SDK.

**Q22.** What does this print?

```dart
class A{ A(){ print('A'); } }
class B extends A{ B(){ print('B'); } }
void main(){ B(); }
```

**Answer:** A
B

The super constructor runs first. Verified by running it with the Dart SDK.

**Q23.** What does `this.name` in a constructor parameter list do?

**Answer:** Assigns the argument directly to the field

Concise initialization.

**Q24.** What does a leading underscore on a name mean?

**Answer:** Library-private: not visible outside the library

No `private` keyword.

### [07 · Null Safety Basics](level-1/07-null-safety-basics.md)

**Q25.** What does this print?

```dart
void main(){ String? s; print(s?.length); print(s ?? 'none'); s ??= 'x'; print(s); }
```

**Answer:** null
none
x

?. ?? and ??= operators. Verified by running it with the Dart SDK.

**Q26.** What does `String name` mean under sound null safety?

**Answer:** It can never be null

String? allows null.

**Q27.** What does the `!` operator do?

**Answer:** Asserts a nullable value isn't null and throws if it is

Use sparingly.

**Q28.** What does `late` mean?

**Answer:** The variable is initialized later but non-null; accessing early throws

Use carefully.

### [08 · Async Basics](level-1/08-async-basics.md)

**Q29.** What does this print?

```dart
Future<int> f() async { await Future.delayed(Duration(milliseconds:10)); return 42; }
void main() async { print('start'); print(await f()); print('end'); }
```

**Answer:** start
42
end

await pauses the async function. Verified by running it with the Dart SDK.

**Q30.** What type does an async function return?

**Answer:** A Future

Mark it async and await results.

**Q31.** Is Dart single-threaded?

**Answer:** Each isolate runs on one thread with an event loop

Isolates have separate memory.

**Q32.** What does `Future.wait` do?

**Answer:** Waits for several futures to complete

Returns a list of results.

### [09 · Packages (pub, pubspec.yaml)](level-1/09-packages.md)

**Q33.** What file lists a project's dependencies?

**Answer:** pubspec.yaml

pub get resolves them.

**Q34.** What does `dart pub add http` do?

**Answer:** Adds the package to pubspec.yaml and fetches it

From pub.dev.

**Q35.** What is pubspec.lock for?

**Answer:** Pinning the exact resolved versions

Commit it for apps, not libraries.

**Q36.** What does `^1.2.3` mean?

**Answer:** Compatible versions &gt;=1.2.3 &lt;2.0.0

Caret constraint.

### [10 · Project — CLI To-Do App](level-1/10-project-todo-cli.md)

**Q37.** How can the CLI persist tasks between runs?

**Answer:** Write them to a JSON file and load on start

dart:io File.

**Q38.** Which collection holds the task list?

**Answer:** List&lt;Task&gt;

Ordered and growable.

**Q39.** How do you read command-line arguments?

**Answer:** main(List&lt;String&gt; args)

Parse the command.

**Q40.** Why validate user input?

**Answer:** To avoid crashes on bad values like non-numeric ids

int.tryParse.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

**Q41.** What does this print?

```dart
abstract class Shape{ double area(); }
class Sq extends Shape{ double s; Sq(this.s); double area()=>s*s; }
void main(){ Shape x = Sq(3); print(x.area()); }
```

**Answer:** 9.0

Abstract classes define a contract. Verified by running it with the Dart SDK.

**Q42.** What does this print?

```dart
mixin Walker{ String move()=>'walk'; }
class Dog with Walker{}
void main(){ print(Dog().move()); }
```

**Answer:** walk

Mixins add reusable behavior. Verified by running it with the Dart SDK.

**Q43.** What does `implements` do?

**Answer:** Requires the class to provide the whole interface of another

Every class is an implicit interface.

**Q44.** What's a factory constructor?

**Answer:** A constructor that may return an existing or subtype instance

Useful for caching/singletons.

### [02 · Null Safety Advanced](level-2/02-null-safety-advanced.md)

**Q45.** What does this print?

```dart
void main(){ int? x = 5; if (x != null) { print(x + 1); } String? s; print(s?.toUpperCase() ?? 'empty'); }
```

**Answer:** 6
empty

Flow analysis promotes x to int. Verified by running it with the Dart SDK.

**Q46.** What does type promotion mean?

**Answer:** After a null check, the compiler treats the variable as non-null

Works for local variables.

**Q47.** Why can't a nullable field be promoted?

**Answer:** It may change between check and use, so copy it to a local first

Use a local variable.

**Q48.** What does `required` do for a named parameter?

**Answer:** Forces callers to pass it

Avoids nulls for important params.

### [03 · Streams](level-2/03-streams.md)

**Q49.** What does this print?

```dart
Stream<int> count() async* { for (var i=1;i<=3;i++) { yield i; } }
void main() async { await for (final v in count()) { print(v*10); } }
```

**Answer:** 10
20
30

async* yields a Stream. Verified by running it with the Dart SDK.

**Q50.** What does this print?

```dart
void main() async { var total = await Stream.fromIterable([1,2,3,4]).where((x)=>x.isEven).fold<int>(0,(a,b)=>a+b); print(total); }
```

**Answer:** 6

where then fold. Verified by running it with the Dart SDK.

**Q51.** What's the difference between a single-subscription and broadcast stream?

**Answer:** Single allows one listener; broadcast allows many

asBroadcastStream.

**Q52.** What does `yield` do in async*?

**Answer:** Emits a value to the stream

Generators.

### [04 · Testing with the test package](level-2/04-testing.md)

**Q53.** Which function defines a test case in package:test?

**Answer:** test('description', () { ... })

group() organizes tests.

**Q54.** What does `expect(actual, matcher)` do?

**Answer:** Asserts the value satisfies the matcher

e.g. equals, throwsA.

**Q55.** How do you run all tests?

**Answer:** dart test

Files end with _test.dart.

**Q56.** How do you test code that throws?

**Answer:** expect(() =&gt; f(), throwsA(isA&lt;FormatException&gt;()))

Wrap in a closure.

### [05 · Working with JSON](level-2/05-json.md)

**Q57.** What does this print?

```dart
import 'dart:convert';
void main(){ var m = jsonDecode('{"a":1,"b":[2,3]}'); print(m['b'][1]); print(jsonEncode({'x':true,'y':null})); }
```

**Answer:** 3
{"x":true,"y":null}

jsonDecode yields Maps and Lists. Verified by running it with the Dart SDK.

**Q58.** What do fromJson factories do?

**Answer:** Build a typed object from a decoded Map

Pair with toJson.

**Q59.** Why type-check decoded JSON?

**Answer:** jsonDecode returns dynamic, and malformed data causes runtime errors

Validate fields.

**Q60.** What does `jsonEncode` accept?

**Answer:** Maps, Lists, numbers, strings, bools, null (or objects with toJson)

Custom classes need toJson.

### [06 · Generics](level-2/06-generics.md)

**Q61.** What does this print?

```dart
class Box<T>{ T v; Box(this.v); T get(){ return v; } }
T first<T>(List<T> l)=>l.first;
void main(){ print(Box<int>(7).get()); print(first(['a','b'])); }
```

**Answer:** 7
a

Generics keep type safety. Verified by running it with the Dart SDK.

**Q62.** What does `<T extends num>` do?

**Answer:** Restricts T to num or its subtypes

Bounded generics.

**Q63.** What's reified generics in Dart?

**Answer:** Generic types are available at runtime

Unlike Java erasure.

**Q64.** Why use List&lt;int&gt; instead of List&lt;dynamic&gt;?

**Answer:** Compile-time type checking

Avoid dynamic.

### [07 · Extension Methods](level-2/07-extension-methods.md)

**Q65.** What does this print?

```dart
extension StringX on String { String get shout => toUpperCase()+'!'; }
void main(){ print('hi'.shout); }
```

**Answer:** HI!

Extensions add members to existing types. Verified by running it with the Dart SDK.

**Q66.** When are extension methods resolved?

**Answer:** Statically, based on the declared type

Not virtual.

**Q67.** Can extensions access private members of the type?

**Answer:** No, only the public API

They don't change the class.

**Q68.** What do extensions help avoid?

**Answer:** Static helper functions and wrapper classes

Readable API.

### [08 · Error Handling](level-2/08-error-handling.md)

**Q69.** What does this print?

```dart
class AppError implements Exception{ final String m; AppError(this.m); String toString()=>'AppError: $m'; }
void main(){ try { throw AppError('bad'); } on AppError catch(e) { print(e); } finally { print('done'); } }
```

**Answer:** AppError: bad
done

finally always runs. Verified by running it with the Dart SDK.

**Q70.** What's the difference between Exception and Error?

**Answer:** Exceptions are recoverable conditions; Errors indicate programmer bugs

Don't catch Errors generally.

**Q71.** What does `rethrow` do?

**Answer:** Rethrows the caught exception preserving the stack trace

Better than throw e.

**Q72.** How do you handle errors from a Future?

**Answer:** try/catch around await, or .catchError

Unhandled async errors crash.

### [09 · Collections Deep Dive](level-2/09-collections-deep-dive.md)

**Q73.** What does this print?

```dart
void main(){ var n=[1,2,3,4]; print(n.where((x)=>x>1).map((x)=>x*x).toList()); print(n.fold<int>(0,(a,b)=>a+b)); print(n.reduce((a,b)=>a>b?a:b)); }
```

**Answer:** [4, 9, 16]
10
4

where, map, fold and reduce. Verified by running it with the Dart SDK.

**Q74.** What does this print?

```dart
void main(){ var l=[[1,2],[3],[4,5]]; print(l.expand((x)=>x).toList()); }
```

**Answer:** [1, 2, 3, 4, 5]

expand flattens. Verified by running it with the Dart SDK.

**Q75.** Are map/where lazy?

**Answer:** Yes: they return lazy Iterables until toList()

Beware repeated evaluation.

**Q76.** What does `sort((a,b)=>b.compareTo(a))` do?

**Answer:** Sorts descending

Comparator.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** Which package makes HTTP requests?

**Answer:** package:http

Also dart:io HttpClient.

**Q78.** How should you handle a non-200 response?

**Answer:** Check statusCode and throw/handle a meaningful error

Don't assume success.

**Q79.** Where should an API key come from?

**Answer:** Environment variables, not source code

Never commit secrets.

**Q80.** Why fetch several cities concurrently?

**Answer:** Future.wait overlaps network waits

I/O bound.

## Level 3 · Advanced

### [01 · Flutter Fundamentals](level-3/01-flutter-fundamentals.md)

**Q81.** In Flutter, what is a widget?

**Answer:** An immutable description of part of the UI; Flutter builds elements and render objects from it

UI is a tree of widgets.

**Q82.** What's the difference between StatelessWidget and StatefulWidget?

**Answer:** Stateful widgets hold mutable State that triggers rebuilds via setState

Use stateless when possible.

**Q83.** What does setState() do?

**Answer:** Marks the widget dirty so build() runs again with new state

Only for local state.

**Q84.** Why is build() expected to be cheap and side-effect free?

**Answer:** It can be called many times per second

Move work out.

### [02 · Building APIs with `shelf`](level-3/02-building-apis-shelf.md)

**Q85.** How does shelf model a server?

**Answer:** A function from Request to Response, composed with middleware

Handler = FutureOr&lt;Response&gt; Function(Request).

**Q86.** What is middleware in shelf?

**Answer:** A function wrapping a handler to add behavior like logging or auth

Pipeline().addMiddleware().

**Q87.** How do you route requests in shelf?

**Answer:** Use shelf_router or inspect request.url

Router().get('/x', handler).

**Q88.** Which status code for a missing resource?

**Answer:** 404

Response.notFound.

### [03 · Databases](level-3/03-databases.md)

**Q89.** Why use parameterized queries with sqlite3?

**Answer:** To prevent SQL injection

Use ? placeholders.

**Q90.** What does sqlite3 give a Dart service?

**Answer:** An embedded SQL database without a server process

File-based.

**Q91.** What does a transaction ensure?

**Answer:** All-or-nothing changes

BEGIN/COMMIT/ROLLBACK.

**Q92.** Why close database handles?

**Answer:** To release resources and flush data

dispose().

### [04 · Design Patterns in Dart](level-3/04-design-patterns.md)

**Q93.** What does this print?

```dart
class Cfg{ static final Cfg _i = Cfg._(); Cfg._(); factory Cfg()=>_i; }
void main(){ print(identical(Cfg(), Cfg())); }
```

**Answer:** true

A factory constructor returning one instance implements Singleton. Verified by running it with the Dart SDK.

**Q94.** What does the Observer pattern provide?

**Answer:** Subjects notify subscribers of changes (Streams in Dart)

StreamController.

**Q95.** What's the Factory pattern for?

**Answer:** Centralizing object creation behind an interface

Choose subtype by input.

**Q96.** What's the Repository pattern?

**Answer:** An abstraction over data access

Swap implementations in tests.

### [05 · Isolates](level-3/05-isolates.md)

**Q97.** What does this print?

```dart
import 'dart:isolate';
int heavy(int n){ var s=0; for (var i=1;i<=n;i++){ s+=i; } return s; }
void main() async { print(await Isolate.run(()=>heavy(1000))); }
```

**Answer:** 500500

Isolate.run executes in a separate isolate. Verified by running it with the Dart SDK.

**Q98.** Why use isolates?

**Answer:** To run CPU-heavy work in parallel without blocking the event loop

Each has separate memory.

**Q99.** How do isolates communicate?

**Answer:** By message passing (ports), not shared memory

Messages are copied.

**Q100.** Is async/await parallel execution?

**Answer:** No: it's concurrency on one thread; use isolates for parallelism

Event loop.

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

**Q101.** What's a mock/fake used for?

**Answer:** Replacing dependencies like HTTP clients in unit tests

mockito or hand-written fakes.

**Q102.** What does setUp() do?

**Answer:** Runs before each test in a group

tearDown is after.

**Q103.** What does test coverage show?

**Answer:** Which lines executed, not assertion quality

dart test --coverage.

**Q104.** How do you test async code?

**Answer:** Make the test async and await, using expectLater for streams

Await futures.

### [07 · State Management Concepts](level-3/07-state-management-concepts.md)

**Q105.** What's the core idea of state management?

**Answer:** Keep state in one place and let UI react to changes

Lift state up.

**Q106.** What's wrong with putting everything in setState?

**Answer:** State shared across widgets becomes hard to manage and test

Use a dedicated approach.

**Q107.** What does ChangeNotifier do?

**Answer:** Notifies listeners when state changes

Used by Provider.

**Q108.** What does unidirectional data flow mean?

**Answer:** State flows down, events flow up

Predictable updates.

### [08 · Performance &amp; Profiling](level-3/08-performance-profiling.md)

**Q109.** What does the Dart DevTools profiler show?

**Answer:** CPU, memory and frame timing to find bottlenecks

Profile in profile mode.

**Q110.** Why profile in release/profile mode, not debug?

**Answer:** Debug mode has extra checks and no AOT optimizations

Timing differs.

**Q111.** What's a common Flutter jank cause?

**Answer:** Expensive work in build() or the UI thread

Use const and isolates.

**Q112.** What does `const` widget constructors enable?

**Answer:** Reuse of identical instances to skip rebuilds

Cheaper builds.

### [09 · Code Generation](level-3/09-code-generation.md)

**Q113.** What does build_runner do?

**Answer:** Runs code generators like json_serializable

dart run build_runner build.

**Q114.** Why generate JSON serialization code?

**Answer:** Avoids error-prone hand-written boilerplate

Annotations drive it.

**Q115.** Are generated files committed?

**Answer:** Depends on policy; keep them consistent and re-generate when sources change

Some projects ignore them.

**Q116.** What do annotations like @JsonSerializable do?

**Answer:** Mark classes for the generator

Metadata.

### [10 · Project — REST API + Database Service](level-3/10-project-rest-api-db.md)

**Q117.** What layers does the API project separate?

**Answer:** Routes/handlers, service logic and data access

Separation of concerns.

**Q118.** Which verb creates a resource?

**Answer:** POST

Return 201 with Location.

**Q119.** How should validation errors respond?

**Answer:** 400 with a clear message

Client errors.

**Q120.** How do you test the API?

**Answer:** Call handlers with Request objects or use an integration test against a running server

Automate.

## Level 4 · Master

### [01 · Advanced Async Patterns](level-4/01-advanced-async-patterns.md)

**Q121.** What does this print?

```dart
void main() async { var r = await Future.wait([Future.value(1), Future.delayed(Duration(milliseconds:5),()=>2)]); print(r); }
```

**Answer:** [1, 2]

Future.wait preserves order. Verified by running it with the Dart SDK.

**Q122.** What's a StreamTransformer for?

**Answer:** Transforming a stream's events (map, debounce, etc.)

Composable pipelines.

**Q123.** Why handle backpressure in streams?

**Answer:** Producers may outpace consumers and exhaust memory

Pause subscriptions.

**Q124.** What does a Completer do?

**Answer:** Lets you complete a Future manually

Bridge callback APIs.

### [02 · Production Dart Services](level-4/02-production-dart-services.md)

**Q125.** What should a production service expose?

**Answer:** Health checks, structured logs and metrics

Operability.

**Q126.** How should configuration be supplied?

**Answer:** Environment variables or config files, not hard-coded

12-factor.

**Q127.** Why handle shutdown signals gracefully?

**Answer:** To finish in-flight requests and close resources

SIGTERM.

**Q128.** What does AOT compilation provide?

**Answer:** Fast startup and a standalone executable

dart compile exe.

### [03 · Flutter State Management at Scale](level-4/03-flutter-state-management-scale.md)

**Q129.** What's a benefit of Bloc/Riverpod-style architecture at scale?

**Answer:** Testable, predictable state separate from UI

Choose consistently.

**Q130.** What should repositories handle?

**Answer:** Data sources behind an abstraction for the state layer

Layering.

**Q131.** Why avoid putting business logic in widgets?

**Answer:** It's hard to test and reuse

Keep widgets dumb.

**Q132.** What does dependency injection enable here?

**Answer:** Swapping implementations for tests

Providers.

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

**Q133.** Where should API secrets live in a Flutter app?

**Answer:** Not embedded in the client; use a backend or secure storage

Clients can be reverse-engineered.

**Q134.** What does certificate pinning defend against?

**Answer:** Man-in-the-middle with fraudulent certificates

Use carefully.

**Q135.** Why validate all input on the server?

**Answer:** Clients can be modified or bypassed

Never trust clients.

**Q136.** How should passwords be stored?

**Answer:** Salted slow hashes (bcrypt/argon2)

Never reversible.

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

**Q137.** What should CI run for a Dart project?

**Answer:** dart analyze, dart format check and dart test

Gate merges.

**Q138.** Why cache the pub cache in CI?

**Answer:** Faster dependency installs

Restore by pubspec.lock hash.

**Q139.** What's a flaky test?

**Answer:** One that passes and fails without code changes

Fix or quarantine.

**Q140.** What does a test pyramid recommend?

**Answer:** Mostly unit tests, fewer integration, fewest end-to-end

Speed and confidence.

### [06 · Deployment with Docker](level-4/06-deployment-docker.md)

**Q141.** Why multi-stage Docker builds for Dart?

**Answer:** Compile to a native binary in a build stage and ship it in a tiny runtime image

Small images.

**Q142.** What does `dart compile exe` produce?

**Answer:** A self-contained native executable

AOT.

**Q143.** Why run as non-root in a container?

**Answer:** Limits damage if compromised

Least privilege.

**Q144.** How should containers receive config?

**Answer:** Environment variables

Inject at runtime.

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

**Q145.** What should you do before optimizing?

**Answer:** Measure with a profiler to find the real bottleneck

Evidence first.

**Q146.** What helps throughput of a Dart server?

**Answer:** Running multiple isolates/instances behind a load balancer

Isolates don't share memory.

**Q147.** Why avoid blocking the event loop?

**Answer:** Everything else on that isolate stalls

Offload CPU work.

**Q148.** What reduces allocations in hot paths?

**Answer:** Reusing objects and using const/typed data

Typed lists.

### [08 · Package Publishing (pub.dev)](level-4/08-package-publishing.md)

**Q149.** What command checks a package before publishing?

**Answer:** dart pub publish --dry-run

Validates layout and metadata.

**Q150.** What versioning does pub use?

**Answer:** Semantic versioning

Breaking changes bump major.

**Q151.** What should a published package include?

**Answer:** README, CHANGELOG, LICENSE, tests and docs

Quality signals on pub.dev.

**Q152.** Can a published version be changed?

**Answer:** No: versions are immutable; publish a new one (or retract)

Immutability.

### [09 · Code Quality (`dart analyze` / lints)](level-4/09-code-quality.md)

**Q153.** What does `dart analyze` do?

**Answer:** Statically checks code for errors and lint violations

Configure analysis_options.yaml.

**Q154.** What does `dart format` do?

**Answer:** Applies the standard code style

Consistent diffs.

**Q155.** Why enable strict lints?

**Answer:** Catch issues early and keep consistency

package:lints or flutter_lints.

**Q156.** What is dead code detection good for?

**Answer:** Removing unused code to reduce maintenance

Analyzer hints.

### [10 · Capstone — Authenticated Notes API](level-4/10-capstone-project.md)

**Q157.** What does the capstone integrate?

**Answer:** API, persistence, testing, CI, containers and quality tooling

End-to-end.

**Q158.** How do you validate the finished project?

**Answer:** Tests, analysis, CI and a containerized run

Layered checks.

**Q159.** What makes it maintainable?

**Answer:** Layering, tests, documentation and consistent style

Sustainable.

**Q160.** What should the README cover?

**Answer:** How to run, test and deploy

Onboarding.

## Mixed questions across all levels

**Q161.** In Dart, what does `String name` mean under sound null safety?

**Answer:** name can never be null; use String? to allow null

Non-nullable by default.

**Q162.** What does this print?

```dart
void main(){ String? s; print(s?.length); print(s ?? 'none'); s ??= 'x'; print(s); }
```

**Answer:** null
none
x

?. short-circuits, ?? gives a default, ??= assigns if null. Verified by running it with the Dart SDK.

**Q163.** What does this print?

```dart
void main(){ final a=[1,2]; a.add(3); print(a); const b=[1,2]; print(b.length); }
```

**Answer:** [1, 2, 3]
2

final stops reassignment, not mutation; const makes an immutable compile-time value. Verified by running it with the Dart SDK.

**Q164.** What does this print?

```dart
void main(){ var n=[1,2,3,4]; print(n.where((x)=>x.isOdd).map((x)=>x*10).toList()); }
```

**Answer:** [10, 30]

where filters, map transforms. Verified by running it with the Dart SDK.

**Q165.** What does this print?

```dart
class A{ A(){ print('A'); } }
class B extends A{ B(){ print('B'); } }
void main(){ B(); }
```

**Answer:** A
B

The superclass constructor runs first. Verified by running it with the Dart SDK.

**Q166.** What does this print?

```dart
mixin Walker{ String move()=>'walk'; }
class Dog with Walker{}
void main(){ print(Dog().move()); }
```

**Answer:** walk

Mixins add reusable behavior. Verified by running it with the Dart SDK.

**Q167.** What does this print?

```dart
void main() async { print('a'); await Future.delayed(Duration(milliseconds:5)); print('b'); }
```

**Answer:** a
b

await pauses this function only. Verified by running it with the Dart SDK.

**Q168.** What does this print?

```dart
Stream<int> s() async* { for (var i=1;i<=3;i++) yield i*i; }
void main() async { await for (final v in s()) print(v); }
```

**Answer:** 1
4
9

async* generators emit with yield. Verified by running it with the Dart SDK.

**Q169.** What does this print?

```dart
import 'dart:isolate';
void main() async { print(await Isolate.run(()=>List.generate(5,(i)=>i*i).reduce((a,b)=>a+b))); }
```

**Answer:** 30

CPU-heavy work can run in a separate isolate. Verified by running it with the Dart SDK.

**Q170.** Why isn't async/await the same as parallelism in Dart?

**Answer:** It interleaves work on one thread via the event loop; isolates give real parallelism with separate memory

Use isolates for CPU-bound work.

**Q171.** A Flutter list janks because build() sorts 10,000 items. What's the fix?

**Answer:** Move the work out of build() (compute once or in an isolate) and use const widgets where possible

build() must be cheap.

**Q172.** State is passed through six widgets only to be used at the bottom. What do you use?

**Answer:** A state management approach (e.g. Provider/Riverpod/Bloc) to lift and share state

Reduces prop drilling.

**Q173.** A shelf handler needs logging and auth for every route. How?

**Answer:** Wrap handlers with middleware in a Pipeline

Middleware composes behavior.

**Q174.** A query builds SQL by concatenating a request parameter. What's wrong?

**Answer:** SQL injection risk: use parameterized queries with placeholders

Bind values.

**Q175.** Where must an API secret not be stored in a Flutter app?

**Answer:** In the client bundle, since it can be extracted; keep it on a backend

Clients are untrusted.

**Q176.** What's the smallest reasonable Docker setup for a Dart server?

**Answer:** Compile with `dart compile exe` in a build stage and copy the binary into a minimal runtime image

Small and fast to start.

**Q177.** A package change breaks callers' code. What version bump?

**Answer:** Major, per semantic versioning

Breaking changes require a major.

**Q178.** CI for a Dart repo should at minimum run...

**Answer:** dart analyze, dart format --set-exit-if-changed and dart test

Gate merges automatically.

## Practical coding questions

### Totals and counts with collections

Implement total(items) with fold (sum of qty x price) and countByInitial(items) returning the total quantity per upper-cased first letter of the item name. Print the total to two decimals and the map.

Starter code:

```
class Item {
  final String name;
  final int qty;
  final double price;
  Item(this.name, this.qty, this.price);
}

double total(List<Item> items) {
  // sum qty * price with fold
  return 0;
}

Map<String, int> countByInitial(List<Item> items) {
  // total quantity per upper-cased first letter of the name
  return {};
}

void main() {
  final items = [
    Item('apple', 3, 0.5),
    Item('avocado', 2, 1.25),
    Item('bread', 1, 2.0),
  ];
  print(total(items).toStringAsFixed(2));
  print(countByInitial(items));
}
```

Expected output:

```text
6.00
{A: 5, B: 1}
```

??? success "Show a solution"

    ```
    class Item {
      final String name;
      final int qty;
      final double price;
      Item(this.name, this.qty, this.price);
    }

    double total(List<Item> items) =>
        items.fold(0.0, (sum, i) => sum + i.qty * i.price);

    Map<String, int> countByInitial(List<Item> items) {
      final counts = <String, int>{};
      for (final i in items) {
        final key = i.name[0].toUpperCase();
        counts[key] = (counts[key] ?? 0) + i.qty;
      }
      return counts;
    }

    void main() {
      final items = [
        Item('apple', 3, 0.5),
        Item('avocado', 2, 1.25),
        Item('bread', 1, 2.0),
      ];
      print(total(items).toStringAsFixed(2));
      print(countByInitial(items));
    }
    ```

    fold accumulates a running sum, and a map with `?? 0` accumulates per-key totals. Dart prints maps as {A: 5, B: 1}. Verified by running the solution with the Dart SDK.

### Typed exceptions and streams

Complete Account.withdraw (ArgumentError for non-positive amounts, a custom InsufficientFunds exception when over balance), and an async* generator ticks(n) that yields 1..n. The provided main handles each failure type and uses finally.

Starter code:

```
import 'dart:async';

class Account {
  int _balance;
  Account(this._balance);
  int get balance => _balance;
  void withdraw(int amount) {
    // throw ArgumentError for amount <= 0, InsufficientFunds(shortBy) when amount > balance,
    // otherwise subtract it from the balance
  }
}

class InsufficientFunds implements Exception {
  final int shortBy;
  InsufficientFunds(this.shortBy);
  @override
  String toString() => 'InsufficientFunds(short by $shortBy)';
}

Stream<int> ticks(int n) async* {
  // yield 1..n
}

Future<void> main() async {
  final acct = Account(100);
  for (final amt in [30, 100, -5]) {
    try {
      acct.withdraw(amt);
      print('withdrew $amt, balance ${acct.balance}');
    } on InsufficientFunds catch (e) {
      print('error: $e');
    } on ArgumentError catch (e) {
      print('invalid: ${e.message}');
    } finally {
      print('checked $amt');
    }
  }
  final total = await ticks(4).fold<int>(0, (a, b) => a + b);
  print('sum of ticks: $total');
}
```

Expected output:

```text
withdrew 30, balance 70
checked 30
error: InsufficientFunds(short by 30)
checked 100
invalid: amount must be positive
checked -5
sum of ticks: 10
```

??? success "Show a solution"

    ```
    import 'dart:async';

    class Account {
      int _balance;
      Account(this._balance);
      int get balance => _balance;
      void withdraw(int amount) {
        if (amount <= 0) throw ArgumentError('amount must be positive');
        if (amount > _balance) throw InsufficientFunds(amount - _balance);
        _balance -= amount;
      }
    }

    class InsufficientFunds implements Exception {
      final int shortBy;
      InsufficientFunds(this.shortBy);
      @override
      String toString() => 'InsufficientFunds(short by $shortBy)';
    }

    Stream<int> ticks(int n) async* {
      for (var i = 1; i <= n; i++) {
        yield i;
      }
    }

    Future<void> main() async {
      final acct = Account(100);
      for (final amt in [30, 100, -5]) {
        try {
          acct.withdraw(amt);
          print('withdrew $amt, balance ${acct.balance}');
        } on InsufficientFunds catch (e) {
          print('error: $e');
        } on ArgumentError catch (e) {
          print('invalid: ${e.message}');
        } finally {
          print('checked $amt');
        }
      }
      final total = await ticks(4).fold<int>(0, (a, b) => a + b);
      print('sum of ticks: $total');
    }
    ```

    Catch the most specific exception types first, use finally for always-run code, and async* with yield to produce a Stream. Verified by running the solution with the Dart SDK.

---

More free practice: the [Dart Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
