# 10 · Project — REST API + Database Service

This project combines the last two modules — [shelf](02-building-apis-shelf.md)
and [databases](03-databases.md) — into one small, real service: a Task API
backed by SQLite, with routing, validation, and a service layer that's
testable independent of HTTP.

```yaml
# pubspec.yaml
dependencies:
  shelf: ^1.4.0
  shelf_router: ^1.1.4
  sqlite3: ^2.4.0
```

## Design: separate the database logic from the HTTP layer

`TaskService` knows nothing about `shelf`, `Request`, or JSON — it's plain
Dart working against a `Database`. That split means it can be unit tested
directly (no server, no HTTP client) and reused if the API layer ever
changes.

```dart
// lib/task_service.dart
import 'package:sqlite3/sqlite3.dart';

class TaskService {
  TaskService(this._db) {
    _db.execute('''
      CREATE TABLE IF NOT EXISTS tasks (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        done INTEGER NOT NULL DEFAULT 0
      );
    ''');
  }

  final Database _db;

  Map<String, dynamic> _rowToJson(Row row) => {
        'id': row['id'],
        'title': row['title'],
        'done': row['done'] == 1, // SQLite has no bool type -- stored as 0/1
      };

  List<Map<String, dynamic>> listTasks() {
    final rows = _db.select('SELECT id, title, done FROM tasks ORDER BY id');
    return rows.map(_rowToJson).toList();
  }

  Map<String, dynamic> createTask(String title) {
    final stmt = _db.prepare('INSERT INTO tasks (title, done) VALUES (?, 0)');
    stmt.execute([title]);
    stmt.dispose();
    final id = _db.lastInsertRowId;
    return {'id': id, 'title': title, 'done': false};
  }

  Map<String, dynamic>? completeTask(int id) {
    final stmt = _db.prepare('UPDATE tasks SET done = 1 WHERE id = ?');
    stmt.execute([id]);
    stmt.dispose();
    if (_db.updatedRows == 0) return null; // no row matched that id
    final row = _db.select('SELECT id, title, done FROM tasks WHERE id = ?', [id]).first;
    return _rowToJson(row);
  }
}
```

`_db.updatedRows` after an `UPDATE` is how `completeTask` distinguishes "id
doesn't exist" from "id exists" — SQLite happily runs an `UPDATE` that
matches zero rows without raising any error, so you have to check the
affected-row count yourself if that distinction matters.

## The HTTP layer: routing, parsing, validating

`buildHandler` wires `TaskService` into three routes, doing just enough
request parsing and validation to keep bad input from ever reaching the
service layer.

```dart
// lib/task_api.dart
import 'dart:convert';
import 'package:shelf/shelf.dart';
import 'package:shelf_router/shelf_router.dart';
import 'task_service.dart';

Handler buildHandler(TaskService service) {
  final router = Router();

  router.get('/tasks', (Request req) {
    return Response.ok(jsonEncode(service.listTasks()), headers: {'content-type': 'application/json'});
  });

  router.post('/tasks', (Request req) async {
    final body = jsonDecode(await req.readAsString()) as Map<String, dynamic>;
    final title = body['title'];
    if (title is! String || title.isEmpty) {
      return Response(400, body: jsonEncode({'error': 'title is required'}));
    }
    final created = service.createTask(title);
    return Response(201, body: jsonEncode(created), headers: {'content-type': 'application/json'});
  });

  router.post('/tasks/<id>/complete', (Request req, String id) {
    final parsedId = int.tryParse(id);
    if (parsedId == null) {
      return Response(400, body: jsonEncode({'error': 'invalid id'}));
    }
    final updated = service.completeTask(parsedId);
    if (updated == null) {
      return Response.notFound(jsonEncode({'error': 'task not found'}));
    }
    return Response.ok(jsonEncode(updated), headers: {'content-type': 'application/json'});
  });

  return const Pipeline().addMiddleware(logRequests()).addHandler(router.call);
}
```

`int.tryParse(id)` (not `int.parse`) matters: a route param is always a
`String`, and a malformed one (`/tasks/abc/complete`) would otherwise throw
a `FormatException` deep inside the handler instead of returning a clean
400.

## Wiring it together and exercising it end to end

```dart
// bin/server.dart
import 'dart:convert';
import 'dart:io';
import 'package:shelf/shelf_io.dart' as shelf_io;
import 'package:sqlite3/sqlite3.dart';
import 'package:task_api/task_api.dart';
import 'package:task_api/task_service.dart';

Future<void> main() async {
  final db = sqlite3.openInMemory();
  final service = TaskService(db);
  final server = await shelf_io.serve(buildHandler(service), 'localhost', 8099);
  print('Task API running on http://${server.address.host}:${server.port}');
}
```

## Running it

Driving the running server with real HTTP calls end to end:

```
Task API running on http://localhost:8099
GET /tasks -> 200 []
POST /tasks -> 201 {"id":1,"title":"Write docs","done":false}
POST /tasks -> 201 {"id":2,"title":"Ship release","done":false}
POST /tasks -> 400 {"error":"title is required"}
POST /tasks/1/complete -> 200 {"id":1,"title":"Write docs","done":true}
POST /tasks/99/complete -> 404 {"error":"task not found"}
GET /tasks -> 200 [{"id":1,"title":"Write docs","done":true},{"id":2,"title":"Ship release","done":false}]
```

## Testing the service layer directly

Because `TaskService` doesn't know about HTTP, its tests need no server at
all — just an in-memory database, exactly like [module 03](03-databases.md):

```dart
// test/task_service_test.dart
import 'package:sqlite3/sqlite3.dart';
import 'package:test/test.dart';
import 'package:task_api/task_service.dart';

void main() {
  late Database db;
  late TaskService service;

  setUp(() {
    db = sqlite3.openInMemory();
    service = TaskService(db);
  });

  tearDown(() => db.dispose());

  test('starts empty', () {
    expect(service.listTasks(), isEmpty);
  });

  test('createTask then listTasks shows it', () {
    service.createTask('Write docs');
    final tasks = service.listTasks();
    expect(tasks, hasLength(1));
    expect(tasks.first['title'], 'Write docs');
    expect(tasks.first['done'], isFalse);
  });

  test('completeTask marks done and returns updated row', () {
    final created = service.createTask('Ship release');
    final updated = service.completeTask(created['id'] as int);
    expect(updated!['done'], isTrue);
  });

  test('completeTask returns null for unknown id', () {
    expect(service.completeTask(999), isNull);
  });
}
```

```
00:00 +4: All tests passed!
```

Four tests, no HTTP server started, no port bound — this is the payoff of
keeping `TaskService` free of `shelf` types: fast, deterministic tests for
the logic that actually matters, with the HTTP layer left thin enough that
it barely needs testing of its own beyond the end-to-end run above.

## Stretch goals

- Add `DELETE /tasks/<id>` and a `deleteTask` method on `TaskService`;
  return 404 for an id that doesn't exist, using the same
  `_db.updatedRows`-style check pattern as `completeTask`.
- Add a `PATCH /tasks/<id>` route that can update a task's `title`,
  validating that the new title is non-empty before writing it.
- Persist to a real file (`sqlite3.open('tasks.db')`) instead of
  `openInMemory()`, and confirm tasks survive a process restart.
- Add pagination to `GET /tasks` via `?limit=&offset=` query parameters
  (`request.url.queryParameters`), defaulting sensibly when absent.
- Wrap the whole handler in a middleware that catches any uncaught
  exception and returns a JSON 500 instead of letting `shelf` return its
  default plain-text error page.
