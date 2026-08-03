# 09 · Collections Deep Dive

[Level 1](../level-1/05-collections.md) covered `List`, `Set`, and `Map`
basics — creating them, indexing, and looping. This module covers Dart's
functional collection methods: `map`, `where`, `fold`, `reduce`, `expand`,
and `sort`, the tools that let you transform and summarize data in a few
expressive lines instead of hand-written loops with mutable accumulators.

## The core five

```dart
class Employee {
  final String name;
  final String department;
  final double salary;
  Employee(this.name, this.department, this.salary);

  @override
  String toString() => '$name ($department, \$$salary)';
}

void main() {
  final employees = [
    Employee('Ada', 'Engineering', 95000),
    Employee('Grace', 'Engineering', 105000),
    Employee('Alan', 'Research', 88000),
    Employee('Linus', 'Engineering', 120000),
    Employee('Margaret', 'Research', 91000),
  ];

  // where: filter -- keep only matching elements
  final engineers = employees.where((e) => e.department == 'Engineering');
  print(engineers.length); // 3

  // map: transform each element -- lazy, doesn't run until consumed
  final names = employees.map((e) => e.name).toList();
  print(names); // [Ada, Grace, Alan, Linus, Margaret]

  // fold: reduce to a single value, with an explicit starting point
  final totalSalary = employees.fold<double>(0, (sum, e) => sum + e.salary);
  print(totalSalary); // 499000.0

  // reduce: like fold, but starts from the first element (fails on empty!)
  final highestPaid = employees.reduce((a, b) => a.salary > b.salary ? a : b);
  print(highestPaid); // Linus (Engineering, $120000.0)

  // expand: flatten -- map each element to zero or more elements
  final initials = employees.expand((e) => e.name.split('')).take(3).toList();
  print(initials); // [A, d, a]

  // sort: in-place, via a comparator (spread into a fresh list first
  // if you don't want to mutate the original)
  final bySalaryDesc = [...employees]
    ..sort((a, b) => b.salary.compareTo(a.salary));
  print(bySalaryDesc.map((e) => e.name).toList());
  // [Linus, Grace, Ada, Margaret, Alan]
}
```

| Method | Input → Output | Fails on empty? |
|---|---|---|
| `where` | `Iterable<T>` → `Iterable<T>` (subset) | No — returns empty |
| `map` | `Iterable<T>` → `Iterable<R>` (transformed) | No — returns empty |
| `fold(initial, combine)` | `Iterable<T>` → single `R`, starting from `initial` | No — returns `initial` |
| `reduce(combine)` | `Iterable<T>` → single `T`, starting from first element | **Yes** — throws `StateError` |
| `expand` | `Iterable<T>` → `Iterable<R>` (flattened) | No — returns empty |

`fold` vs `reduce` is the one worth memorizing carefully: reach for `fold`
whenever the collection might be empty, or when your accumulator's type
differs from the element type (like folding `List<Employee>` down to a
`double` total above) — `reduce` can't do that, since it has no separate
initial value to fall back to.

## Grouping — a pattern, not a built-in

Dart doesn't ship a `groupBy` in the core library. The standard pattern uses
`Map.putIfAbsent` to build up the groups as you iterate once:

```dart
Map<String, List<Employee>> groupByDepartment(List<Employee> employees) {
  final result = <String, List<Employee>>{};
  for (final e in employees) {
    // putIfAbsent -- create the list only the first time we see this key.
    result.putIfAbsent(e.department, () => []).add(e);
  }
  return result;
}

void main() {
  final employees = [
    Employee('Ada', 'Engineering'),
    Employee('Grace', 'Engineering'),
    Employee('Alan', 'Research'),
  ];

  final grouped = groupByDepartment(employees);
  for (final entry in grouped.entries) {
    print('${entry.key}: ${entry.value.map((e) => e.name).toList()}');
  }
  // Engineering: [Ada, Grace]
  // Research: [Alan]
}
```

## The trap: `map`/`where` are lazy

`map` and `where` don't run their callback immediately — they build a
description of the transformation and only execute it when something
actually consumes the result (`toList()`, a `for` loop, `.length` in some
cases, etc.). This surprises people coming from languages where these
operations eagerly build a new list on the spot.

```dart
void main() {
  int callCount = 0;
  final lazy = [1, 2, 3].map((n) {
    callCount++;
    return n * 2;
  });

  print('right after map(): $callCount'); // 0 -- nothing ran yet!

  final materialized = lazy.toList();
  print('after toList(): $callCount'); // 3
  print(materialized); // [2, 4, 6]

  // Re-iterating a lazy Iterable's VALUES re-runs the transform from
  // scratch -- results are never cached.
  for (final v in lazy) {
    // (just draining it)
  }
  print('after iterating lazy again: $callCount'); // 6, not 3
}
```

Two consequences worth knowing: **(1)** a `map`/`where` chain with a
callback that has side effects (logging, incrementing a counter, mutating
something) will re-run those side effects every time the result is
consumed — call `.toList()` once and reuse the list if you need the
callback to run exactly once. **(2)** chaining several lazy operations
(`.where(...).map(...).take(3)`) is efficient precisely because nothing
computes until the final consumption, and `.take(3)` can stop the whole
pipeline early instead of processing every element first.

## Cheat sheet

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

## Exercise

Given a `List<Employee>` like the one above, write one expression (chaining
`where`, `map`, and `fold` or `reduce`) that computes the **average salary of
just the Engineering department**, and a second function `Map<String,
double> averageSalaryByDepartment(List<Employee> employees)` that returns
the average salary for *every* department, built using the
`groupByDepartment` pattern above plus a `.map()` over the resulting
`Map`'s entries.
