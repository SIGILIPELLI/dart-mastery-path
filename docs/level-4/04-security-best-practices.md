# 04 · Security Best Practices

[Databases](../level-3/03-databases.md) covered SQL injection. This module
covers the rest of the security fundamentals every production Dart service
needs: hashing secrets properly, using a cryptographically secure random
source, and validating file paths built from user input.

```yaml
# pubspec.yaml
dependencies:
  crypto: ^3.0.3
```

## The trap: plain hashing is not password storage

`sha256` alone is fast and deterministic — exactly the two properties you
don't want for password storage. Fast means an attacker with a stolen hash
database can try billions of guesses per second; deterministic means two
users with the same password get the identical hash, leaking that fact.

```dart
import 'dart:convert';
import 'package:crypto/crypto.dart';

String hashPasswordInsecure(String password) {
  return sha256.convert(utf8.encode(password)).toString();
}

void main() {
  const password = 'correct horse battery staple';
  print('Insecure hash: ${hashPasswordInsecure(password)}');
  print('Insecure hash again (identical!): ${hashPasswordInsecure(password)}');
}
// Insecure hash: c4bbcb1fbec99d65bf59d85c8cb62ee2db963f0fe106f483d9afa73bd4e39a8a
// Insecure hash again (identical!): c4bbcb1fbec99d65bf59d85c8cb62ee2db963f0fe106f483d9afa73bd4e39a8a
```

Real systems add a per-user random **salt** (so identical passwords hash
differently) and use a deliberately *slow* algorithm (bcrypt, scrypt,
Argon2 — not covered by `package:crypto`, which only provides fast general-
purpose hashes) so brute-forcing is expensive even per-guess. Salting alone,
shown here with plain SHA-256, only fixes the "identical passwords, identical
hashes" half of the problem — it's a stepping stone to understanding the
real fix, not a production-ready scheme by itself.

```dart
import 'dart:math';

String generateSalt() {
  final random = Random.secure(); // cryptographically secure, unlike Random()
  final bytes = List<int>.generate(16, (_) => random.nextInt(256));
  return base64Url.encode(bytes);
}

String hashPasswordSalted(String password, String salt) {
  return sha256.convert(utf8.encode('$salt:$password')).toString();
}

void main() {
  const password = 'correct horse battery staple';
  final saltA = generateSalt();
  final saltB = generateSalt();
  print('Salted hash A: ${hashPasswordSalted(password, saltA)}');
  print('Salted hash B (different salt, same password): ${hashPasswordSalted(password, saltB)}');
  print('Salts differ: ${saltA != saltB}');
}
// Salted hash A: 3bc54580d987720268eb0ab16f230c334808755495cdc8ca264f50b54337c84b
// Salted hash B (different salt, same password): fb1c14c29b7df168182434e8f66555f74533b44924ff18e560c7c1128858a651
// Salts differ: true
```

## The trap: `Random()` is not cryptographically secure

Plain `Random()` is a fast pseudo-random generator meant for things like
game logic and shuffling — its output is predictable enough that an
attacker who sees a few outputs can potentially reconstruct future ones.
Anything security-sensitive (salts, tokens, session ids, password reset
codes) must use `Random.secure()`, which draws from the OS's
cryptographically secure entropy source. The only visible difference in the
API is the constructor name — which makes reaching for the wrong one an
easy, silent mistake.

## Path traversal: never trust a user-supplied path

Serving files based on user input (a filename in a URL, an upload path)
without validation lets `../` sequences escape the directory you meant to
restrict access to.

```dart
import 'dart:io';

Future<void> main() async {
  // ... publicDir contains hello.txt; a sibling secret.txt sits outside it.

  // DANGER: joining user input directly onto a base path lets '..' escape it.
  String unsafeResolve(String base, String userPath) => '$base/$userPath';

  final userInput = '../secret.txt';
  final resolvedUnsafe = File(unsafeResolve(publicDir.path, userInput)).absolute.path;
  print('Unsafe read: ${await File(resolvedUnsafe).readAsString()}');
}
// Unsafe read: TOP SECRET
```

The fix: normalize the resulting path and explicitly check that it's still
inside the allowed root before touching the filesystem.

```dart
String? safeResolve(String base, String userPath) {
  final baseAbs = Directory(base).absolute.path;
  final candidate = Uri.file('$base/$userPath').normalizePath().toFilePath();
  if (!candidate.startsWith('$baseAbs${Platform.pathSeparator}') && candidate != baseAbs) {
    return null; // escaped the allowed root -- reject
  }
  return candidate;
}

void main() {
  print(safeResolve(publicDir.path, '../secret.txt')); // null -- rejected
  print(safeResolve(publicDir.path, 'hello.txt'));      // resolves normally
}
// null
// /tmp/.../public/hello.txt
```

`Uri.file(...).normalizePath()` is the key step — without it, `..`
segments are still present in the string and a naive `startsWith` check on
the un-normalized path can be fooled by the very sequence it's trying to
detect.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `sha256` alone for passwords | Fast + deterministic — wrong properties for password storage |
| Salt (`Random.secure()` bytes) | Makes identical passwords hash differently |
| bcrypt / scrypt / Argon2 | Deliberately slow — what real password hashing should use (not `sha256`) |
| `Random()` | Predictable — fine for games, unsafe for tokens/salts |
| `Random.secure()` | Cryptographically secure source — use for anything security-sensitive |
| Path traversal (`../`) | User input joined onto a path can escape the intended directory |
| `Uri.file(path).normalizePath()` | Collapses `..`/`.` before validating the path is still in-bounds |

## Exercise

Write a function `String? safeJoin(String root, String userPath)` that
combines the normalize-then-verify pattern above into a single reusable
helper, rejecting both `../` traversal attempts and absolute paths (a
`userPath` starting with `/` that would otherwise bypass `root` entirely).
Test it against: a normal relative file, a `../../etc/passwd`-style
traversal, and an absolute path like `/etc/passwd` passed directly as
`userPath` — all three should be handled correctly (one allowed, two
rejected).
