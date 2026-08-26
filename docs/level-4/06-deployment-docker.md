# 06 · Deployment with Docker

!!! note "Docker itself wasn't run for this module"
    No Docker daemon was available in the environment these lessons were
    written in. The Dart side — compiling to a native executable and
    running it with a container-style environment variable — was actually
    run and verified below; the Dockerfile follows Dart's documented,
    widely-used multi-stage pattern and was reviewed carefully rather than
    executed with `docker build`.

[Production Dart services](02-production-dart-services.md) covered
configuring from the environment and shutting down gracefully — exactly
what you need once that service runs inside a container. This module
covers `dart compile exe` and packaging the result into a minimal Docker
image.

## Compiling to a standalone executable

`dart compile exe` produces a single native binary with the Dart runtime
embedded — no separate Dart SDK install needed to run it, which is exactly
what makes it a good fit for a minimal container.

```dart
// bin/server.dart
import 'dart:io';

Future<void> main() async {
  final port = int.tryParse(Platform.environment['PORT'] ?? '') ?? 8080;
  final server = await HttpServer.bind(InternetAddress.anyIPv4, port);
  print('Listening on 0.0.0.0:$port');
  await for (final request in server) {
    request.response
      ..statusCode = 200
      ..write('ok')
      ..close();
  }
}
```

```bash
dart compile exe bin/server.dart -o bin/server
PORT=8093 ./bin/server &
curl localhost:8093
```

```
Listening on 0.0.0.0:8093
ok
```

`InternetAddress.anyIPv4` (not `localhost`/`127.0.0.1`) matters inside a
container: a server bound only to `localhost` is unreachable from outside
the container's own network namespace, even with the port correctly
published — a very common "works locally, unreachable when containerized"
bug.

## A multi-stage Dockerfile

Building in one stage and running in another keeps the final image small —
the SDK, source code, and build cache never make it into the image that
actually ships.

```dockerfile
# Stage 1: build the executable using the full Dart SDK image
FROM dart:stable AS build
WORKDIR /app

COPY pubspec.* ./
RUN dart pub get

COPY . .
RUN dart pub get --offline
RUN dart compile exe bin/server.dart -o bin/server

# Stage 2: minimal runtime image with just the compiled binary
FROM scratch
COPY --from=build /runtime/ /
COPY --from=build /app/bin/server /app/bin/server

EXPOSE 8080
ENV PORT=8080
ENTRYPOINT ["/app/bin/server"]
```

`FROM scratch` (an empty base image) plus `COPY --from=build /runtime/ /` —
copying the minimal runtime files the official Dart image documents at that
path — produces an image containing essentially nothing but the compiled
binary and what it needs to run, typically tens of megabytes rather than
the ~1GB+ of a full SDK image.

## The trap: `COPY pubspec.* ./` before `COPY . .` isn't just style

Splitting dependency installation from copying the rest of the source is a
deliberate Docker layer-caching optimization, not an arbitrary ordering.
Docker caches each `RUN`/`COPY` layer and only re-runs a layer (and
everything after it) if its inputs changed.

```dockerfile
# Slower: any source change invalidates the pub get cache too
COPY . .
RUN dart pub get
RUN dart compile exe bin/server.dart -o bin/server
```

```dockerfile
# Faster: pub get only reruns when pubspec.yaml/pubspec.lock actually change
COPY pubspec.* ./
RUN dart pub get
COPY . .
RUN dart compile exe bin/server.dart -o bin/server
```

With the first ordering, editing a single line of application code
invalidates every layer including `dart pub get` — a full dependency
re-resolve on every build. With the second, `pub get`'s layer stays cached
across ordinary code changes and only reruns when `pubspec.yaml` or
`pubspec.lock` actually changes — the difference between a rebuild taking
seconds versus a minute or more once a project has real dependencies.

## Cheat sheet

| Concept | Meaning |
|---|---|
| `dart compile exe` | Produces a standalone native binary, no SDK needed to run it |
| `InternetAddress.anyIPv4` | Bind to all interfaces — required to be reachable from outside a container |
| Multi-stage Dockerfile | Build with the full SDK image, ship only the compiled binary |
| `FROM scratch` + Dart's `/runtime/` | Minimal final image — tens of MB instead of a full SDK |
| `COPY pubspec.* ./` before `COPY . .` | Caches `pub get` separately so code edits don't force a re-resolve |
| `ENV PORT` + reading it in the app | Standard way to make the container's port configurable at run time |

## Exercise

Write a `Dockerfile` for the Task API project from
[Level 3's capstone](../level-3/10-project-rest-api-db.md), using the
multi-stage pattern above, that reads `PORT` from the environment and binds
to `InternetAddress.anyIPv4`. Add a `HEALTHCHECK` instruction that curls
`/tasks` (add `curl` in the build stage only if needed — note that `FROM
scratch` has no shell or curl available, so explain in a comment what
tradeoff that forces if you want a working `HEALTHCHECK`, and pick
`google/dart:stable-slim` or a `distroless` base instead if you want one).
