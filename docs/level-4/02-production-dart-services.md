# 02 · Production Dart Services

[Building APIs with shelf](../level-3/02-building-apis-shelf.md) got a
server running. Running that server *in production* needs three more
things this module covers: configuration from the environment (not
hardcoded values), structured logging that a log aggregator can parse, and
graceful shutdown so in-flight requests finish instead of getting dropped
when the process is stopped.

## Configuration from environment variables

Hardcoded ports and hosts don't survive contact with a real deployment —
containers, staging vs. production, and secrets all flow through
environment variables. `Platform.environment` is Dart's read-only view of
the process environment.

```dart
class Config {
  Config({required this.port, required this.logLevel});

  final int port;
  final String logLevel;

  factory Config.fromEnv(Map<String, String> env) {
    final portStr = env['PORT'];
    final port = portStr != null ? int.tryParse(portStr) ?? 8080 : 8080;
    final logLevel = env['LOG_LEVEL'] ?? 'info';
    return Config(port: port, logLevel: logLevel);
  }
}
```

Taking `Map<String, String> env` as a parameter — rather than reading
`Platform.environment` directly inside the factory — is the detail that
matters: tests can now call `Config.fromEnv({'PORT': '9000'})` without
touching real process state at all.

## Structured (JSON) logging

A production service's logs are read by machines (log aggregators,
alerting rules) as often as by humans — one JSON object per line is the
standard format, versus `print()`'s unstructured text.

```dart
import 'dart:convert';
import 'dart:io';

void logJson(String level, String message, [Map<String, dynamic>? fields]) {
  final entry = {
    'timestamp': DateTime.now().toUtc().toIso8601String(),
    'level': level,
    'message': message,
    ...?fields, // spread additional structured fields if provided
  };
  stdout.writeln(jsonEncode(entry));
}
```

```
{"timestamp":"2026-08-26T16:49:22.313734Z","level":"info","message":"listening","port":8087}
```

Using UTC (`.toUtc()`) for the timestamp avoids a whole category of bugs
where logs from servers in different timezones (or a server vs. the
aggregator collecting them) can't be correlated by time.

## Graceful shutdown on `SIGTERM`

Container orchestrators (Docker, Kubernetes) stop a process by sending
`SIGTERM` and then, after a grace period, `SIGKILL` if it hasn't exited.
`ProcessSignal.sigterm.watch()` lets you catch the first signal and shut
down cleanly — close the listening socket, let in-flight requests finish,
then exit — instead of connections being severed mid-response.

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

void logJson(String level, String message, [Map<String, dynamic>? fields]) {
  final entry = {
    'timestamp': DateTime.now().toUtc().toIso8601String(),
    'level': level,
    'message': message,
    ...?fields,
  };
  stdout.writeln(jsonEncode(entry));
}

Future<void> main() async {
  final server = await HttpServer.bind('localhost', 8087);
  logJson('info', 'listening', {'port': server.port});
  server.listen((request) {
    request.response
      ..statusCode = 200
      ..write('ok')
      ..close();
  });

  final done = Completer<void>();
  late StreamSubscription sub;
  sub = ProcessSignal.sigterm.watch().listen((signal) async {
    logJson('info', 'shutdown initiated', {'signal': signal.toString()});
    await sub.cancel();
    await server.close(force: false); // let pending requests finish
    logJson('info', 'shutdown complete');
    done.complete();
  });

  await done.future;
}
```

Compiled to a native executable and sent a real `SIGTERM` from another
process:

```
{"timestamp":"2026-08-26T16:49:22.313734Z","level":"info","message":"listening","port":8087}
{"timestamp":"2026-08-26T16:49:22.675617Z","level":"info","message":"shutdown initiated","signal":"SIGTERM"}
{"timestamp":"2026-08-26T16:49:22.675982Z","level":"info","message":"shutdown complete"}
```

## The trap: `ProcessSignal.watch()` doesn't work the same everywhere

`ProcessSignal.sigterm.watch()` and `.sigint.watch()` work on Linux and
macOS. `sigterm` specifically is **not supported on Windows** (it throws a
`SignalException` if you try to watch it there) — only `sigint` (Ctrl+C) is.
A service meant to run cross-platform needs to either only rely on
`sigint`, or wrap the `sigterm` watch in a `try`/`catch` and accept that
graceful shutdown on `SIGTERM` simply isn't available on that platform.
This is also why `dart run` (which runs your script inside a wrapper
process/daemon for hot-reload support) can behave differently under signals
than a `dart compile exe` binary receiving them directly — always verify
signal handling against the actual deployed artifact, not just `dart run`.

## Health checks

A minimal but real production requirement: an endpoint an orchestrator can
poll to know the service is alive and ready to receive traffic.

```dart
Response healthCheck(Request request) {
  return Response.ok(jsonEncode({'status': 'ok', 'uptime_s': _uptime.elapsedSeconds}));
}

final _uptime = Stopwatch()..start();
extension on Stopwatch {
  int get elapsedSeconds => elapsed.inSeconds;
}
```

Wire this at `GET /health` alongside the real routes from
[module 02 of Level 3](../level-3/02-building-apis-shelf.md) — orchestrators
typically distinguish *liveness* (is the process up at all) from
*readiness* (is it able to serve traffic, e.g. is its database connection
established); a single `/health` endpoint is a fine start, two endpoints is
the more complete production pattern.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `Config.fromEnv(Map<String,String>)` | Take env as a parameter, not `Platform.environment` directly — testable |
| JSON-per-line logging | Machine-parseable; include a UTC timestamp on every entry |
| `ProcessSignal.sigterm.watch()` | Catch shutdown signals to close cleanly instead of being killed |
| `server.close(force: false)` | Stop accepting new connections, let in-flight ones finish |
| `sigterm` on Windows | Not supported — only `sigint` works there |
| `dart run` vs `dart compile exe` | Signal delivery can differ — test against the real deployed artifact |
| `/health` endpoint | Minimal liveness/readiness check for orchestrators |

## Exercise

Write a `GracefulServer` class that wraps `HttpServer.bind`, exposes a
`Future<void> waitForShutdown()` that completes when `SIGINT` is received,
and a `Future<void> shutdown()` that closes the server and logs (via
`logJson`) both the start and completion of shutdown. Add a `GET /health`
route that returns `{"status": "ok", "uptime_s": N}` using a `Stopwatch`
started at server creation. Test it manually by running the compiled
executable and sending it Ctrl+C, confirming the shutdown log lines appear
before the process exits.
