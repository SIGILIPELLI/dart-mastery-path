# 02 · Building APIs with `shelf`

[Level 1's async basics](../level-1/08-async-basics.md) and
[Level 2's streams](../level-2/03-streams.md) covered async I/O in the
abstract. `shelf` is the standard Dart package for building HTTP servers:
it models a server as a single function — `Request` in, `Response` out (or
a `Future` of one) — and lets you compose behavior with middleware. This
module builds a small JSON API with it.

```yaml
# pubspec.yaml
dependencies:
  shelf: ^1.4.0
  shelf_router: ^1.1.4
```

## The simplest possible server

A `Handler` is just `FutureOr<Response> Function(Request request)`.
`shelf_io.serve` binds it to a socket and calls it for every incoming
request.

```dart
import 'dart:convert';
import 'dart:io';
import 'package:shelf/shelf.dart';
import 'package:shelf/shelf_io.dart' as shelf_io;

Response _handler(Request request) {
  final name = request.url.path.isEmpty ? 'world' : request.url.path;
  return Response.ok('Hello, $name!');
}

Future<void> main() async {
  final server = await shelf_io.serve(_handler, 'localhost', 8080);
  print('Serving at http://${server.address.host}:${server.port}');

  // A plain HttpClient stands in for a browser/curl request here.
  final client = HttpClient();
  final req = await client.get('localhost', 8080, '/dart');
  final resp = await req.close();
  print('Client got: ${await resp.transform(utf8.decoder).join()}');

  await server.close();
  client.close();
}
// Serving at http://localhost:8080
// Client got: Hello, dart!
```

`request.url.path` is the path **with the leading slash stripped**, so a
request to `/dart` shows up as `'dart'`, not `'/dart'` — easy to trip over
when matching paths by hand.

## Routing with `shelf_router`

Real APIs have more than one endpoint. `shelf_router`'s `Router` maps
HTTP method + path pattern to a handler, including `<param>` placeholders
passed as extra positional arguments.

```dart
import 'dart:convert';
import 'dart:io';
import 'package:shelf/shelf.dart';
import 'package:shelf/shelf_io.dart' as shelf_io;
import 'package:shelf_router/shelf_router.dart';

final _books = <int, String>{1: 'Dart in Action', 2: 'Effective Dart'};

Response _listBooks(Request request) {
  final asStrings = _books.map((k, v) => MapEntry(k.toString(), v));
  return Response.ok(jsonEncode(asStrings), headers: {'content-type': 'application/json'});
}

Response _getBook(Request request, String id) {
  final book = _books[int.parse(id)];
  if (book == null) {
    return Response.notFound(jsonEncode({'error': 'not found'}));
  }
  return Response.ok(jsonEncode({'id': id, 'title': book}), headers: {'content-type': 'application/json'});
}

Future<void> main() async {
  final router = Router()
    ..get('/books', _listBooks)
    ..get('/books/<id>', _getBook);

  final handler = const Pipeline().addMiddleware(logRequests()).addHandler(router.call);
  final server = await shelf_io.serve(handler, 'localhost', 8081);
  print('Serving at http://${server.address.host}:${server.port}');

  final client = HttpClient();
  for (final path in ['/books', '/books/1', '/books/99']) {
    final req = await client.get('localhost', 8081, path);
    final resp = await req.close();
    print('$path -> ${resp.statusCode} ${await resp.transform(utf8.decoder).join()}');
  }
  await server.close();
  client.close();
}
// Serving at http://localhost:8081
// /books -> 200 {"1":"Dart in Action","2":"Effective Dart"}
// /books/1 -> 200 {"id":"1","title":"Dart in Action"}
// /books/99 -> 404 {"error":"not found"}
```

Route parameters like `id` always arrive as `String` — `_getBook` has to
`int.parse` it itself, and a malformed id (`/books/abc`) would throw
`FormatException` inside the handler rather than failing at the routing
layer.

## The trap: `jsonEncode` rejects non-`String` map keys

The very first version of `_listBooks` above used `jsonEncode(_books)`
directly on a `Map<int, String>`. That throws at runtime:

```
Converting object to an encodable object failed: _Map len:2
```

JSON objects only have string keys, and `jsonEncode` does **not**
stringify integer keys for you — it throws instead. This is one of the more
confusing runtime errors in Dart because the map itself is perfectly valid
Dart; it only breaks the moment it meets `jsonEncode`. The fix, shown above,
is to `.map((k, v) => MapEntry(k.toString(), v))` before encoding.

## Middleware: composing cross-cutting behavior

Middleware wraps a handler with a function that runs before/after it —
`logRequests()` above is one; you can write your own the same way, e.g. to
require an API key on every request.

```dart
import 'package:shelf/shelf.dart';

Middleware requireApiKey(String expected) {
  return (Handler inner) {
    return (Request request) {
      final key = request.headers['x-api-key'];
      if (key != expected) {
        return Response.forbidden('missing or invalid API key');
      }
      return inner(request);
    };
  };
}
```

`Pipeline().addMiddleware(a).addMiddleware(b).addHandler(h)` runs `a`'s
"before" logic, then `b`'s, then `h`, then `b`'s "after" logic, then `a`'s —
middleware nests like an onion, not a flat list, which matters once one
middleware's job is to catch errors thrown further in.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `Handler` | `FutureOr<Response> Function(Request)` — the whole server model |
| `shelf_io.serve(handler, host, port)` | Bind a handler to a real socket |
| `Router()..get(path, fn)` | Route by method + path, with `<param>` captures |
| Route params | Always arrive as `String`; parse/validate yourself |
| `Pipeline().addMiddleware(...).addHandler(...)` | Compose cross-cutting behavior around a handler |
| `jsonEncode` on non-`String` map keys | Throws at runtime — stringify keys first |
| `logRequests()` | Built-in middleware that prints one line per request |

## Exercise

Build a `shelf_router` API with two routes: `POST /echo`, which reads the
request body with `await request.readAsString()` and returns it back with
status 200, and `GET /health`, which returns `{"status":"ok"}` as JSON.
Add a middleware that logs the method and path of every request before
`logRequests()` runs. Verify both routes with an `HttpClient` in `main()`,
printing status code and body for each.
