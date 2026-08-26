# 09 · Code Quality (`dart analyze` / lints)

[Advanced async patterns](01-advanced-async-patterns.md) showed a missing
`await` crashing a program with an unhandled exception. This module covers
the tool that catches that mistake — and many others — before the code ever
runs: `dart analyze`, configured with a lint set.

```yaml
# pubspec.yaml (dev_dependencies)
dev_dependencies:
  lints: ^6.1.0
```

## Enabling a recommended lint set

`analysis_options.yaml` at the project root configures the analyzer.
`package:lints` ships two starting presets — `core.yaml` (minimal,
correctness-focused) and `recommended.yaml` (broader, includes style rules)
— that you extend with `include:`, then layer your own rules on top.

```yaml
# analysis_options.yaml
include: package:lints/recommended.yaml

linter:
  rules:
    - unawaited_futures
    - prefer_final_locals
    - avoid_print
```

## Seeing it catch a real bug

```dart
Future<void> risky() async {
  await Future.delayed(const Duration(milliseconds: 10));
}

Future<void> main() async {
  risky(); // missing await -- flagged by unawaited_futures
  var x = 5; // flagged by prefer_final_locals
  print(x); // flagged by avoid_print
}
```

```bash
dart analyze bin/qual1.dart
```

```
info - Missing an 'await' for the 'Future' computed by this expression. Try adding an 'await' or wrapping the expression with 'unawaited'. - unawaited_futures
info - Local variables should be final. Try making the variable final. - prefer_final_locals
info - Don't invoke 'print' in production code. Try using a logging framework. - avoid_print

3 issues found.
```

`unawaited_futures` is the automated version of the missing-`await` trap
from [module 01](01-advanced-async-patterns.md) — the exact bug that turns
a caught exception into a process crash, caught here at analysis time
instead of discovered in production.

## The trap: `unawaited_futures` only fires inside `async` functions

The lint above only triggers when the discarded-future expression sits
inside an `async` function body — a plain (non-`async`) `main()` calling
the same `risky()` the same way produces **zero** lint warnings for it,
even though the bug is identical. The rule's authors made this tradeoff
deliberately: a non-`async` function calling something that returns a
`Future` and not waiting for it is a common, often-intentional
"fire-and-forget" pattern (starting background work from synchronous setup
code), so the lint stays narrowly scoped to where it's almost always a
mistake — inside code that's already writing `await` everywhere else. This
means the lint is a safety net, not a guarantee: reviewing async code by
eye for missing `await`s is still worthwhile, especially at a non-`async`
call site.

## `dart fix`: auto-applying lint corrections

Many lints (though not `unawaited_futures`, which needs a judgment call
about whether the future should actually be awaited or genuinely ignored)
have a mechanical fix the tool can apply for you.

```bash
dart fix --dry-run   # preview what would change
dart fix --apply     # apply all available automatic fixes
```

`prefer_final_locals` above is exactly this kind of case —
`dart fix --apply` would change `var x = 5;` to `final x = 5;`
automatically, no manual edit needed.

## Deliberately silencing a lint

Sometimes a lint is wrong for one specific line — silence it narrowly, with
a comment explaining why, rather than disabling the rule project-wide.

```dart
// ignore: avoid_print
print('deliberate debug output, removed before merge');
```

Suppressing at the project level in `analysis_options.yaml` (`- avoid_print:
false`-style overrides, or excluding whole files) should be rare and
reviewed — a lint disabled globally stops protecting *any* code, including
future code that would have genuinely benefited from the check.

## Cheat sheet

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

## Exercise

Add `avoid_dynamic_calls` and `always_declare_return_types` to your
`analysis_options.yaml`'s rule list. Write a small file that violates both
(a function with no declared return type, and a call through a `dynamic`-
typed variable), run `dart analyze` to confirm both are flagged, then fix
them by hand. Finally, add one deliberate, justified `// ignore:` comment
for a lint you believe is a false positive in that specific spot, and write
a one-sentence justification as the comment text explaining why.
