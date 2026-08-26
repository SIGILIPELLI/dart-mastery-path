# 03 · Databases

[Level 2's JSON module](../level-2/05-json.md) covered talking to APIs.
Sooner or later a Dart service needs to persist data itself. This module
uses the `sqlite3` package — a real embedded SQL database with no server
process, network round-trip, or external service to run — to cover the
patterns (and traps) that carry over to Postgres, MySQL, or any other SQL
database you'll use from a production Dart service.

```yaml
# pubspec.yaml
dependencies:
  sqlite3: ^2.4.0
```

## Opening a database and running SQL

`sqlite3.openInMemory()` gives you a throwaway database that lives only for
the process — perfect for examples and tests. `sqlite3.open('path.db')`
persists to disk instead.

```dart
import 'package:sqlite3/sqlite3.dart';

void main() {
  final db = sqlite3.openInMemory();

  db.execute('''
    CREATE TABLE employees (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      role TEXT NOT NULL
    );
  ''');

  final insert = db.prepare('INSERT INTO employees (name, role) VALUES (?, ?)');
  insert.execute(['Ada', 'Engineer']);
  insert.execute(['Grace', 'Architect']);
  insert.dispose(); // prepared statements hold native resources -- dispose them

  final result = db.select('SELECT id, name, role FROM employees ORDER BY id');
  for (final row in result) {
    print('${row['id']}: ${row['name']} (${row['role']})');
  }

  db.dispose();
}
// 1: Ada (Engineer)
// 2: Grace (Architect)
```

Every value in `sqlite3` — `Database`, `PreparedStatement` — wraps a native
resource and needs an explicit `.dispose()`. There's no garbage-collector
finalizer doing it for you; leaving it out leaks native memory that Dart's
own GC can't see or reclaim.

## The trap: `execute()` interpolation is a real injection hole

`db.select()` and `PreparedStatement.execute()` take a parameter list and
bind values safely. Plain `db.execute(sql)` runs whatever SQL string you
hand it — **with no placeholder binding at all** — so building that string
with interpolated user input lets the input change the statement's
structure, not just its data.

```dart
import 'package:sqlite3/sqlite3.dart';

void main() {
  final db = sqlite3.openInMemory();
  db.execute('CREATE TABLE employees (id INTEGER PRIMARY KEY, name TEXT NOT NULL UNIQUE);');
  db.execute("INSERT INTO employees (name) VALUES ('Ada')");

  final userInput = "Bad'); DROP TABLE employees; --";

  // DANGER: string-building SQL lets user input change the statement's
  // structure, not just its data.
  db.execute("INSERT INTO employees (name) VALUES ('$userInput')");

  try {
    final count = db.select('SELECT COUNT(*) AS c FROM employees').first['c'];
    print('Row count: $count');
  } on SqliteException catch (e) {
    print('Table is gone: ${e.message}');
  }
  db.dispose();
}
// Table is gone: no such table: employees
```

The `employees` table is gone — the interpolated `'); DROP TABLE...` closed
the `INSERT` early and ran a second statement. The fix is always the same:
never build SQL with string interpolation of untrusted values; use `?`
placeholders and pass values as a list instead.

```dart
final stmt = db.prepare('INSERT INTO employees (name) VALUES (?)');
stmt.execute([userInput]); // treated purely as data, never as SQL syntax
stmt.dispose();
// Safe row: Ada
// Safe row: Bad'); DROP TABLE employees; --
```

With a bound parameter, the malicious-looking string is stored as a literal
`name` value — exactly the point of parameterized queries.

## Constraints throw `SqliteException`, not a Dart-level check

Uniqueness, `NOT NULL`, and foreign-key rules are enforced by SQLite itself
and surface as a catchable `SqliteException` — Dart never validates them for
you ahead of time.

```dart
import 'package:sqlite3/sqlite3.dart';

void main() {
  final db = sqlite3.openInMemory();
  db.execute('CREATE TABLE employees (id INTEGER PRIMARY KEY, name TEXT UNIQUE);');
  db.execute("INSERT INTO employees (name) VALUES ('Ada')");

  try {
    db.execute("INSERT INTO employees (name) VALUES ('Ada')");
  } on SqliteException catch (e) {
    print('Caught: ${e.message}');
  }
  db.dispose();
}
// Caught: UNIQUE constraint failed: employees.name
```

Catch `SqliteException` specifically (not a bare `catch`) so unrelated bugs
in your own code don't get silently swallowed alongside real constraint
violations.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `sqlite3.openInMemory()` | Throwaway DB for the process lifetime — good for tests |
| `sqlite3.open('file.db')` | Persistent DB backed by a file |
| `db.execute(sql)` | Run raw SQL, no parameter binding — never interpolate input into it |
| `db.prepare(sql)` / `.execute([...])` | Bind `?` placeholders safely — use for any dynamic value |
| `db.select(sql, [params])` | Run a query and get back rows you can index like a map |
| `.dispose()` | Required on `Database` and `PreparedStatement` — native resources, no GC finalizer |
| `SqliteException` | Thrown for constraint violations and SQL errors — catch it specifically |

## Exercise

Create a `projects` table (`id`, `name TEXT UNIQUE`, `owner TEXT NOT NULL`)
and a function `void addProject(Database db, String name, String owner)`
that inserts a row using a prepared statement, catching `SqliteException`
and printing a friendly `"Project '$name' already exists"` message on a
duplicate `name`. Write a `main()` that creates an in-memory database, adds
three projects (with one deliberate duplicate name), then prints all rows
ordered by `name`.
