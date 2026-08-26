# 05 · Testing at Scale & CI

[Testing advanced](../level-3/06-testing-advanced.md) covered fakes and
async matchers for individual tests. Once a suite has hundreds of tests,
new problems show up: some are slow, some need to run in specific
environments, and you need a way to know a change didn't break anything
before it merges. This module covers test tagging, coverage measurement,
and wiring both into CI.

```yaml
# pubspec.yaml (dev_dependencies)
dev_dependencies:
  test: ^1.25.0
```

## Tagging tests to run subsets

`tags:` on a `test()` lets you group tests by cost or category, then
include/exclude them from a given run — critical once a "slow" integration
suite would otherwise make every local test run take minutes.

```dart
import 'package:test/test.dart';

void main() {
  test('fast unit test', () {
    expect(1 + 1, 2);
  }, tags: ['unit']);

  test('slow integration-style test', () async {
    await Future.delayed(const Duration(milliseconds: 200));
    expect(true, isTrue);
  }, tags: ['slow']);
}
```

Declare tags up front in `dart_test.yaml` (avoids a warning, and lets you
configure per-tag behavior like timeouts):

```yaml
# dart_test.yaml
tags:
  unit:
  slow:
```

```bash
dart test --exclude-tags slow   # fast local loop
dart test --tags slow           # just the slow ones, e.g. nightly CI
```

```
fast unit test
+1: All tests passed!
```

Excluding `slow` skips the integration-style test entirely — it's never
even scheduled to run, not run-and-ignored.

## Measuring coverage

`dart test --coverage=coverage` records which lines executed during the
run; the `coverage` package's `format_coverage` tool turns that into a
standard LCOV report CI dashboards (Codecov, Coveralls) understand.

```bash
dart pub global activate coverage
dart test --coverage=coverage
dart pub global run coverage:format_coverage \
  --lcov --in=coverage --out=coverage/lcov.info --packages=.dart_tool/package_config.json
```

Coverage percentage is a signal, not a target to chase for its own sake — 100%
line coverage with no assertions on behavior (a test that calls a function
and checks nothing) is worthless. Use coverage reports to find code with
**zero** tests touching it, not to optimize a number.

## The trap: coverage numbers hide untested branches

A line can be "covered" while a whole conditional branch inside it never
actually gets exercised.

```dart
String describe(int n) {
  if (n < 0) return 'negative';   // covered if any negative n is tested
  return 'non-negative';           // "covered" the moment any n >= 0 is tested
}
```

A single test calling `describe(5)` reports both lines as "covered" in a
naive line-coverage tool, without a single test ever exercising the
`n < 0` branch. Line coverage answers "did this line execute," not "were
all its meaningful paths exercised" — for branch-level confidence you need
tests written to *deliberately* hit each conditional outcome, which no
coverage percentage substitutes for.

## Wiring it into CI

A minimal GitHub Actions workflow that runs on every push and fails the
build on any test failure:

```yaml
# .github/workflows/test.yml
name: test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dart-lang/setup-dart@v1
      - run: dart pub get
      - run: dart analyze --fatal-infos
      - run: dart test --exclude-tags slow
```

`dart analyze --fatal-infos` runs before the tests and fails the build on
any analyzer issue (including lint-level "info" severity, not just errors)
— catching style and correctness problems for the price of one extra step,
before spending time running the test suite at all.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `tags: ['slow']` on `test()` | Group tests to include/exclude by category |
| `dart_test.yaml` `tags:` block | Declares valid tags, avoids "undeclared tag" warnings |
| `dart test --exclude-tags X` | Skip tagged tests for a fast local loop |
| `dart test --coverage=DIR` | Record which lines executed |
| `format_coverage --lcov` | Convert to LCOV for CI coverage dashboards |
| Line coverage ≠ branch coverage | A "covered" line can still hide an untested conditional path |
| `dart analyze --fatal-infos` in CI | Fail the build on any analyzer issue, not just errors |

## Exercise

Add `tags: ['unit']` to every fast test and `tags: ['integration']` to
anything using `Future.delayed` longer than 100ms across a small test file
of your own. Write a `dart_test.yaml` declaring both tags with a `timeout`
override for `integration` tests (e.g. 30s). Then write a GitHub Actions
matrix job that runs `dart test --tags unit` on every push, and a separate
nightly-scheduled job (`on: schedule`) that runs `dart test --tags
integration` — explain in a comment why splitting them this way keeps the
push-triggered feedback loop fast.
