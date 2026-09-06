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

## How It Actually Works

A plain hash function (`sha256`, `md5`) is designed to be *fast* — that's
its whole job as a cryptographic primitive for integrity checking, and it's
exactly the wrong property for password storage: an attacker with a stolen
hash database can compute billions of candidate hashes per second on
commodity GPU hardware, because there's no algorithmic reason a fast hash
takes longer than a single arithmetic-heavy pass over the input. Password
hashing algorithms (`bcrypt`, `scrypt`, `Argon2`) are deliberately
**slow and memory-hard** by design — they run the underlying mixing function
thousands of times in a loop, and/or require large amounts of RAM per
guess, specifically to make brute-force guessing computationally expensive
per attempt, which a general-purpose fast hash simply never was built to
do. A per-password random salt, stored alongside the hash, defeats
precomputed rainbow-table attacks by forcing an attacker to redo the
expensive hashing work for every single password rather than reusing one
precomputed table across an entire stolen database.

`Random()` (Dart's default, non-seeded constructor) is a standard
pseudorandom number generator optimized for statistical distribution and
speed, using an algorithm whose internal state can, in principle, be
inferred from a sequence of its outputs — fine for shuffling a game board,
unacceptable for anything security-sensitive because an attacker who can
observe or guess enough outputs could predict future ones. `Random.secure()`
instead sources entropy from the operating system's cryptographically
secure random number generator (e.g., `/dev/urandom` on POSIX, `CryptGenRandom`-
family APIs on Windows) — a fundamentally different data source designed to
resist exactly that kind of prediction, at some cost in speed since it may
block briefly on entropy availability at OS level.

Path traversal succeeds because filesystem path resolution happens after
string concatenation — a path like `uploads/../../etc/passwd` is a
syntactically valid relative path, and the OS's own path-resolution logic
(not Dart's) walks the `..` segments up out of the intended directory before
ever checking whether the final resolved path was supposed to be reachable.
The fix (`p.normalize()` plus checking the resolved path stays under an
allowed root, or rejecting `..` outright) has to happen in your own code
specifically because the OS filesystem API has no concept of "the directory
this request was supposed to be confined to" — that's an application-level
invariant.

## Exercise

Write a function `String? safeJoin(String root, String userPath)` that
combines the normalize-then-verify pattern above into a single reusable
helper, rejecting both `../` traversal attempts and absolute paths (a
`userPath` starting with `/` that would otherwise bypass `root` entirely).
Test it against: a normal relative file, a `../../etc/passwd`-style
traversal, and an absolute path like `/etc/passwd` passed directly as
`userPath` — all three should be handled correctly (one allowed, two
rejected).
