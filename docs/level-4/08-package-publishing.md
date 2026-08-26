# 08 · Package Publishing (pub.dev)

[Packages](../level-1/09-packages.md) covered *consuming* packages from
pub.dev. This module covers the other side: structuring, versioning, and
validating a package of your own before publishing it — using
`dart pub publish --dry-run`, which checks everything a real publish would
without actually uploading anything.

## The files pub.dev expects

A publishable package needs a specific, small set of files at its root,
beyond just `lib/`:

```
string_niceties/
├── lib/
│   └── string_niceties.dart
├── test/
│   └── string_niceties_test.dart
├── pubspec.yaml
├── README.md
├── CHANGELOG.md
└── LICENSE
```

```yaml
# pubspec.yaml
name: string_niceties
description: A small collection of ergonomic String extension methods for common formatting tasks.
version: 0.1.0
homepage: https://github.com/example/string_niceties
environment:
  sdk: ^3.0.0
dev_dependencies:
  test: ^1.25.0
  lints: ^4.0.0
```

`description` shows up directly on the pub.dev listing and in search
results — pub.dev's own scoring flags descriptions that are too short or
too long, so it's worth writing a real sentence, not a placeholder.

## Writing the library

```dart
// lib/string_niceties.dart
/// Ergonomic extensions on [String] for common formatting tasks.
library string_niceties;

extension StringNiceties on String {
  /// Returns this string with the first letter capitalized.
  ///
  /// Returns the string unchanged if it's empty.
  String capitalize() {
    if (isEmpty) return this;
    return this[0].toUpperCase() + substring(1);
  }

  /// Truncates to [maxLength], appending [ellipsis] if truncated.
  String truncate(int maxLength, {String ellipsis = '...'}) {
    if (length <= maxLength) return this;
    return substring(0, maxLength) + ellipsis;
  }
}
```

`///` doc comments aren't just for readers browsing the source — pub.dev
renders them directly on the package's generated API reference pages, and
the analyzer surfaces them in IDE tooltips for anyone who depends on the
package.

```dart
// test/string_niceties_test.dart
import 'package:string_niceties/string_niceties.dart';
import 'package:test/test.dart';

void main() {
  test('capitalize', () {
    expect('hello'.capitalize(), 'Hello');
    expect(''.capitalize(), '');
  });

  test('truncate', () {
    expect('hello world'.truncate(5), 'hello...');
    expect('hi'.truncate(5), 'hi');
  });
}
// 00:00 +2: All tests passed!
```

## Validating before publishing

`dart pub publish --dry-run` runs every check the real publish would —
required files, `pubspec.yaml` correctness, package size — without
uploading anything.

```bash
dart pub publish --dry-run
```

```
Total compressed archive size: 18 KB.
Validating package...
The server may enforce additional checks.

Package has 0 warnings.
```

## The trap: pub.dev checks for exact filenames, not near-misses

The required files are matched by **exact name** — `LICENSE`, not
`LICENSE.md` or `LICENSE.txt` (either of those is fine too, actually, but a
typo like `LICENCE` or `LICENSE.bak` is not recognized at all).

```bash
mv LICENSE LICENSE.bak
dart pub publish --dry-run
```

```
Validating package...
Package validation found the following potential issue:
* Please consider renaming ./LICENSE.bak to `LICENSE`. See https://dart.dev/tools/pub/publishing#important-files.
The server may enforce additional checks.

Package has 1 warning.
```

This is a "warning," not a hard failure — pub.dev will still accept the
publish — but a package listed without a detected license shows that
prominently on its pub.dev page and affects its trust/pub score, so it's
worth treating any dry-run warning as something to actually fix, not
ignore.

## Semantic versioning and the `CHANGELOG`

Dart's package ecosystem depends on every package following semver
(`MAJOR.MINOR.PATCH`): increment `PATCH` for bug fixes, `MINOR` for
backward-compatible new features, `MAJOR` for breaking changes. Every
published version needs a matching entry at the top of `CHANGELOG.md` —
pub.dev renders it on the package page, and it's often the only thing a
consumer reads before deciding whether upgrading is safe.

```markdown
## 0.1.0

- Initial release: `capitalize()` and `truncate()` extensions on `String`.
```

## Cheat sheet

| File/step | Purpose |
|---|---|
| `pubspec.yaml` `description` | Shown on pub.dev listing/search — write a real sentence |
| `README.md` | Rendered as the pub.dev package page's main content |
| `CHANGELOG.md` | One entry per published version — consumers read this before upgrading |
| `LICENSE` (exact name) | Required for pub.dev's license detection and trust score |
| `dart pub publish --dry-run` | Runs every publish check without uploading |
| Doc comments (`///`) | Rendered as the generated API reference on pub.dev |
| Semver (`MAJOR.MINOR.PATCH`) | `PATCH` = fix, `MINOR` = compatible addition, `MAJOR` = breaking change |

## Exercise

Add a third extension method, `String wordCount()`, returning the number of
whitespace-separated words (0 for an empty or whitespace-only string). Add
a corresponding test, bump the package version to `0.2.0` per semver
(a new backward-compatible feature), and add a matching `## 0.2.0` entry at
the top of `CHANGELOG.md` describing the addition. Run
`dart pub publish --dry-run` again and confirm it still reports 0 warnings.
