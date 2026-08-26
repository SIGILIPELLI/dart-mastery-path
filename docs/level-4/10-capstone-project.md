# 10 · Capstone — Authenticated Notes API

This capstone pulls together the whole level: [shelf routing](../level-3/02-building-apis-shelf.md)
and [databases](../level-3/03-databases.md) from Level 3, plus
[security](04-security-best-practices.md) (salted password hashing, secure
tokens), [production logging](02-production-dart-services.md), and
middleware-based auth from this level. It's a small but real service: users
sign up, log in, and manage their own private notes.

```yaml
# pubspec.yaml
dependencies:
  shelf: ^1.4.0
  shelf_router: ^1.1.4
  sqlite3: ^2.4.0
  crypto: ^3.0.3
```

## Auth: salted hashes and bearer tokens

Following [module 04](04-security-best-practices.md): every password gets
its own random salt, and a successful login mints an opaque random token
(not a real JWT — no signing, no expiry — a deliberate simplification
called out in the stretch goals) mapped to a user id in memory.

```dart
import 'dart:convert';
import 'dart:math';
import 'package:crypto/crypto.dart';
import 'package:sqlite3/sqlite3.dart';

String _generateSalt() {
  final random = Random.secure();
  return base64Url.encode(List<int>.generate(16, (_) => random.nextInt(256)));
}

String _hash(String password, String salt) =>
    sha256.convert(utf8.encode('$salt:$password')).toString();

class AuthService {
  AuthService(this._db) {
    _db.execute('''
      CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT NOT NULL UNIQUE,
        salt TEXT NOT NULL,
        password_hash TEXT NOT NULL
      );
    ''');
  }

  final Database _db;
  final Map<String, int> _tokens = {}; // token -> userId

  ({int id, String error})? signUp(String username, String password) {
    final salt = _generateSalt();
    final hash = _hash(password, salt);
    try {
      final stmt = _db.prepare(
          'INSERT INTO users (username, salt, password_hash) VALUES (?, ?, ?)');
      stmt.execute([username, salt, hash]);
      stmt.dispose();
      return (id: _db.lastInsertRowId, error: '');
    } on SqliteException {
      return (id: -1, error: 'username taken'); // UNIQUE constraint hit
    }
  }

  String? logIn(String username, String password) {
    final rows = _db.select('SELECT id, salt, password_hash FROM users WHERE username = ?', [username]);
    if (rows.isEmpty) return null;
    final row = rows.first;
    if (_hash(password, row['salt'] as String) != row['password_hash']) return null;
    final token = base64Url.encode(List<int>.generate(24, (_) => Random.secure().nextInt(256)));
    _tokens[token] = row['id'] as int;
    return token;
  }

  int? userIdForToken(String? token) => token == null ? null : _tokens[token];
}
```

Catching `SqliteException` specifically around the insert (rather than
checking for an existing username first) avoids a race between "check" and
"insert" — the database's own `UNIQUE` constraint is the actual source of
truth for uniqueness, same lesson as [module 03 of Level 3](../level-3/03-databases.md).

## Notes: private per-user data

```dart
class NoteService {
  NoteService(this._db) {
    _db.execute('''
      CREATE TABLE IF NOT EXISTS notes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        body TEXT NOT NULL
      );
    ''');
  }

  final Database _db;

  Map<String, dynamic> createNote(int userId, String body) {
    final stmt = _db.prepare('INSERT INTO notes (user_id, body) VALUES (?, ?)');
    stmt.execute([userId, body]);
    stmt.dispose();
    return {'id': _db.lastInsertRowId, 'body': body};
  }

  List<Map<String, dynamic>> notesFor(int userId) {
    final rows = _db.select('SELECT id, body FROM notes WHERE user_id = ? ORDER BY id', [userId]);
    return rows.map((r) => {'id': r['id'], 'body': r['body']}).toList();
  }
}
```

Every query filters by `user_id` — there is no "get all notes" route.
Forgetting that `WHERE user_id = ?` on any future route would leak every
user's notes to every other user; it's the single most important line in
this whole service.

## Middleware: authenticating every `/notes` request in one place

Rather than checking the token inside every notes handler, one middleware
wraps the whole `/notes` sub-router and stores the authenticated user id on
`Request.context` for handlers to read.

```dart
import 'package:shelf/shelf.dart';

Middleware requireAuth(AuthService auth) {
  return (Handler inner) {
    return (Request request) {
      final header = request.headers['authorization'];
      final token = header?.startsWith('Bearer ') == true ? header!.substring(7) : null;
      final userId = auth.userIdForToken(token);
      if (userId == null) {
        return Response(401, body: jsonEncode({'error': 'unauthorized'}));
      }
      return inner(request.change(context: {'userId': userId}));
    };
  };
}
```

```dart
final notesRouter = Router()
  ..get('/', (Request req) {
    final userId = req.context['userId'] as int;
    return Response.ok(jsonEncode(notes.notesFor(userId)));
  })
  ..post('/', (Request req) async {
    final userId = req.context['userId'] as int;
    final body = jsonDecode(await req.readAsString()) as Map<String, dynamic>;
    return Response(201, body: jsonEncode(notes.createNote(userId, body['body'] as String)));
  });

router.mount('/notes', const Pipeline().addMiddleware(requireAuth(auth)).addHandler(notesRouter.call));
```

`router.mount` attaches a whole sub-router (with its own middleware
pipeline) under a path prefix — every route inside `notesRouter` gets
`requireAuth` applied without repeating it per-route.

## Running it

Driving the running server end to end — sign up, a duplicate signup, log
in, an unauthenticated request, then an authenticated create-and-list:

```
{"timestamp":"...","level":"info","message":"listening","port":8098}
POST /signup -> 201 {"id":1}
POST /signup -> 409 {"error":"username taken"}
POST /login -> 200 {"token":"F9Ak5uHMVmDCUMs_u_7o4OGsZ44tSpAK"}
GET /notes -> 401 {"error":"unauthorized"}
GET /notes -> 200 []
POST /notes -> 201 {"id":1,"body":"Buy milk"}
GET /notes -> 200 [{"id":1,"body":"Buy milk"}]
```

The unauthenticated `GET /notes` is rejected by `requireAuth` before ever
reaching `NoteService`; the authenticated calls after login see an
initially-empty list, then the note they just created — end-to-end proof
that auth, storage, and per-user scoping all work together correctly.

## Stretch goals

- Replace the in-memory `_tokens` map with a `sessions` table (token,
  user_id, expires_at) so tokens survive a process restart and can actually
  expire — as-is, every token is invalidated the moment the process
  restarts, and none of them ever expire on their own.
- Add rate limiting to `/login` (e.g. a `Map<String, int>` of failed
  attempts per username with a cooldown) to blunt brute-force password
  guessing.
- Add `PATCH /notes/<id>` and `DELETE /notes/<id>`, making sure both check
  `user_id` matches the authenticated user — not just that the note id
  exists — before allowing the edit or delete.
- Package it with the [Docker module's](06-deployment-docker.md)
  multi-stage pattern, reading `PORT` from the environment and binding to
  `InternetAddress.anyIPv4`.
- Add a `dart_test.yaml`-tagged test suite ([module 05](05-testing-at-scale-ci.md))
  covering `AuthService` and `NoteService` directly against an in-memory
  database, plus a GitHub Actions workflow that runs it on every push.
- Swap the hand-rolled `sha256`-with-salt hashing for a real password
  hashing library (bcrypt/Argon2 via a suitable package) as flagged in
  [module 04](04-security-best-practices.md) — this capstone's hashing is
  good enough to demonstrate the *pattern*, not production-grade on its own.
