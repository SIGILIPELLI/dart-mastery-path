# 10 · Project — Weather CLI

Time to combine everything from this level — [streams](03-streams.md),
[JSON](05-json.md), [error handling](08-error-handling.md), and
[testing](04-testing.md) — into one real program: a command-line tool that
takes one or more city names and prints their current weather, fetched live
from a public API.

## What we're building

A CLI tool that:

- accepts any number of city names as command-line arguments
- looks up each city's coordinates via a free geocoding API
- fetches current weather for each city **concurrently**, streaming results
  back as each one finishes (not waiting for the slowest before showing any)
- reports a clear per-city error (bad city name, API failure) without
  letting one failure take down the others
- ships with an offline test suite that doesn't depend on network access

We'll use [Open-Meteo](https://open-meteo.com/), a free weather API that
needs **no API key and no signup** — ideal for a learning project you can
actually run right now.

## Project setup

```bash
dart create weather_cli
cd weather_cli
dart pub add http
dart pub add --dev test
```

`http` is the standard package for making HTTP requests in Dart (`dart:io`
has a lower-level `HttpClient`, but `package:http` is simpler and also works
on web, unlike `dart:io`). Final `pubspec.yaml` dependencies section:

```yaml
dependencies:
  http: ^1.6.0

dev_dependencies:
  test: ^1.31.2
```

## Project layout

```
weather_cli/
├── bin/
│   └── weather_cli.dart          # entry point
├── lib/
│   ├── models/
│   │   ├── geocoding_result.dart # a resolved city location
│   │   └── weather_report.dart   # a parsed current-weather snapshot
│   ├── weather_exceptions.dart   # custom exception types
│   ├── weather_service.dart      # talks to the two Open-Meteo endpoints
│   └── weather_stream.dart       # fetches many cities concurrently
├── test/
│   ├── weather_report_test.dart
│   └── weather_service_test.dart
└── pubspec.yaml
```

## The models

`GeocodingResult` represents one match from the geocoding API; `WeatherReport`
is the final, displayable result, with a helper that turns Open-Meteo's
numeric [WMO weather codes](https://open-meteo.com/en/docs) into readable text.

```dart
// lib/models/geocoding_result.dart
class GeocodingResult {
  final String name;
  final String country;
  final double latitude;
  final double longitude;

  GeocodingResult({
    required this.name,
    required this.country,
    required this.latitude,
    required this.longitude,
  });

  factory GeocodingResult.fromJson(Map<String, dynamic> json) {
    return GeocodingResult(
      name: json['name'] as String,
      country: json['country'] as String? ?? 'Unknown',
      latitude: (json['latitude'] as num).toDouble(),
      longitude: (json['longitude'] as num).toDouble(),
    );
  }
}
```

```dart
// lib/models/weather_report.dart

/// Maps Open-Meteo's numeric WMO weather codes to a human-readable
/// description. Not every code is listed -- unknowns fall back gracefully.
String describeWeatherCode(int code) {
  const descriptions = {
    0: 'Clear sky',
    1: 'Mainly clear',
    2: 'Partly cloudy',
    3: 'Overcast',
    45: 'Fog',
    48: 'Depositing rime fog',
    51: 'Light drizzle',
    53: 'Moderate drizzle',
    55: 'Dense drizzle',
    61: 'Slight rain',
    63: 'Moderate rain',
    65: 'Heavy rain',
    71: 'Slight snow fall',
    73: 'Moderate snow fall',
    75: 'Heavy snow fall',
    80: 'Slight rain showers',
    81: 'Moderate rain showers',
    82: 'Violent rain showers',
    95: 'Thunderstorm',
  };
  return descriptions[code] ?? 'Unknown conditions (code $code)';
}

class WeatherReport {
  final String city;
  final String country;
  final double temperatureCelsius;
  final double windSpeedKph;
  final int humidityPercent;
  final int weatherCode;

  WeatherReport({
    required this.city,
    required this.country,
    required this.temperatureCelsius,
    required this.windSpeedKph,
    required this.humidityPercent,
    required this.weatherCode,
  });

  /// Builds a report from the geocoding result plus the raw "current"
  /// object returned by Open-Meteo's forecast endpoint.
  factory WeatherReport.fromApi({
    required String city,
    required String country,
    required Map<String, dynamic> current,
  }) {
    return WeatherReport(
      city: city,
      country: country,
      temperatureCelsius: (current['temperature_2m'] as num).toDouble(),
      windSpeedKph: (current['wind_speed_10m'] as num).toDouble(),
      humidityPercent: (current['relative_humidity_2m'] as num).toInt(),
      weatherCode: (current['weather_code'] as num).toInt(),
    );
  }

  String get description => describeWeatherCode(weatherCode);

  @override
  String toString() {
    return '$city, $country: ${temperatureCelsius.toStringAsFixed(1)}°C, '
        '$description, humidity $humidityPercent%, '
        'wind ${windSpeedKph.toStringAsFixed(1)} km/h';
  }
}
```

## Custom exceptions

Following [Module 8](08-error-handling.md)'s pattern: distinct exception
types for distinct failure causes, so callers can react to each
appropriately instead of pattern-matching error message strings.

```dart
// lib/weather_exceptions.dart

/// Thrown when the geocoding API has no match for a city name.
class CityNotFoundException implements Exception {
  final String city;
  CityNotFoundException(this.city);

  @override
  String toString() => 'CityNotFoundException: no location found for "$city"';
}

/// Thrown when the weather API itself fails (bad status code, network
/// error, or unexpected response shape).
class WeatherApiException implements Exception {
  final String message;
  WeatherApiException(this.message);

  @override
  String toString() => 'WeatherApiException: $message';
}
```

## The service — two API calls, chained

`WeatherService` accepts an optional `http.Client` in its constructor
specifically so tests can substitute a fake one instead of hitting the real
network (see the test file below).

```dart
// lib/weather_service.dart
import 'dart:convert';
import 'package:http/http.dart' as http;

import 'models/geocoding_result.dart';
import 'models/weather_report.dart';
import 'weather_exceptions.dart';

class WeatherService {
  final http.Client _client;

  WeatherService({http.Client? client}) : _client = client ?? http.Client();

  static const _geocodingUrl = 'https://geocoding-api.open-meteo.com/v1/search';
  static const _forecastUrl = 'https://api.open-meteo.com/v1/forecast';

  /// Looks up a city name and returns its best-match location.
  Future<GeocodingResult> _geocode(String city) async {
    final uri = Uri.parse(_geocodingUrl).replace(queryParameters: {
      'name': city,
      'count': '1',
      'language': 'en',
      'format': 'json',
    });

    final response = await _client.get(uri);
    if (response.statusCode != 200) {
      throw WeatherApiException(
        'Geocoding request failed with status ${response.statusCode}',
      );
    }

    final body = jsonDecode(response.body) as Map<String, dynamic>;
    final results = body['results'] as List<dynamic>?;
    if (results == null || results.isEmpty) {
      throw CityNotFoundException(city);
    }

    return GeocodingResult.fromJson(results.first as Map<String, dynamic>);
  }

  /// Fetches the current weather for an already-resolved location.
  Future<WeatherReport> _fetchCurrentWeather(GeocodingResult location) async {
    final uri = Uri.parse(_forecastUrl).replace(queryParameters: {
      'latitude': location.latitude.toString(),
      'longitude': location.longitude.toString(),
      'current': 'temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code',
      'timezone': 'auto',
    });

    final response = await _client.get(uri);
    if (response.statusCode != 200) {
      throw WeatherApiException(
        'Forecast request failed with status ${response.statusCode}',
      );
    }

    final body = jsonDecode(response.body) as Map<String, dynamic>;
    final current = body['current'] as Map<String, dynamic>?;
    if (current == null) {
      throw WeatherApiException('Forecast response missing "current" data');
    }

    return WeatherReport.fromApi(
      city: location.name,
      country: location.country,
      current: current,
    );
  }

  /// Public entry point: city name in, weather report out.
  Future<WeatherReport> getWeather(String city) async {
    final location = await _geocode(city);
    return _fetchCurrentWeather(location);
  }

  void close() => _client.close();
}
```

## Streaming multiple cities concurrently

This is where [Module 3](03-streams.md)'s stream knowledge pays off. Instead
of awaiting each city one at a time (slow — total time is the *sum* of every
request) or using `Future.wait` (which only reports results once *every*
city is done), `Stream.fromFutures` yields each result **the moment its own
request finishes** — a fast city's weather shows up immediately, without
waiting for a slower one.

Each per-city future also catches its own error and resolves to a
`WeatherResult` either way, so one bad city name can never take down the
whole stream — it just arrives as a failure result instead of a success.

```dart
// lib/weather_stream.dart
import 'dart:async';

import 'models/weather_report.dart';
import 'weather_service.dart';

/// A result wrapper so the stream can report a per-city failure without
/// ending the whole stream -- one bad city name shouldn't stop the rest.
class WeatherResult {
  final String city;
  final WeatherReport? report;
  final Object? error;

  WeatherResult.success(this.city, WeatherReport this.report) : error = null;
  WeatherResult.failure(this.city, Object this.error) : report = null;

  bool get isSuccess => report != null;
}

/// Fetches weather for several cities concurrently and yields each
/// [WeatherResult] as soon as its request finishes -- results can arrive
/// out of order, since a fast city's response doesn't wait for a slow one.
Stream<WeatherResult> fetchWeatherForCities(
  WeatherService service,
  List<String> cities,
) {
  final futures = cities.map((city) async {
    try {
      final report = await service.getWeather(city);
      return WeatherResult.success(city, report);
    } catch (e) {
      return WeatherResult.failure(city, e);
    }
  });

  return Stream.fromFutures(futures);
}
```

## The CLI entry point

```dart
// bin/weather_cli.dart
import 'package:weather_cli/weather_exceptions.dart';
import 'package:weather_cli/weather_service.dart';
import 'package:weather_cli/weather_stream.dart';

Future<void> main(List<String> args) async {
  if (args.isEmpty) {
    print('Usage: dart run bin/weather_cli.dart <city> [<city> ...]');
    print('Example: dart run bin/weather_cli.dart London Tokyo "New York"');
    return;
  }

  final service = WeatherService();

  try {
    var successCount = 0;
    var failureCount = 0;

    await for (final result in fetchWeatherForCities(service, args)) {
      if (result.isSuccess) {
        print(result.report);
        successCount++;
      } else {
        final error = result.error;
        final reason = switch (error) {
          CityNotFoundException e => e.toString(),
          WeatherApiException e => e.toString(),
          _ => 'Unexpected error: $error',
        };
        print('Could not get weather for "${result.city}" -- $reason');
        failureCount++;
      }
    }

    print('\n$successCount succeeded, $failureCount failed.');
  } finally {
    service.close(); // always release the HTTP client's resources
  }
}
```

## Running it

```bash
dart run bin/weather_cli.dart London Tokyo "Nowhere Fakeplace 12345"
```

```
Could not get weather for "Nowhere Fakeplace 12345" -- CityNotFoundException: no location found for "Nowhere Fakeplace 12345"
London, United Kingdom: 19.4°C, Clear sky, humidity 72%, wind 12.2 km/h
Tokyo, Japan: 25.9°C, Overcast, humidity 74%, wind 6.1 km/h

2 succeeded, 1 failed.
```

Notice the order: the invalid city fails fastest (one API call, immediately
rejected), so its failure prints before either real city's weather — direct,
visible proof that results are streaming back as they complete, not in the
order the arguments were given.

## Testing it without hitting the network

Real unit tests shouldn't depend on a live API being reachable, fast, or
rate-limit-friendly. `package:http` ships a `MockClient` (from
`package:http/testing.dart`) specifically so `WeatherService` — built above
to accept any `http.Client` — can be tested against **fake** responses.

```dart
// test/weather_service_test.dart
import 'package:http/http.dart' as http;
import 'package:http/testing.dart';
import 'package:test/test.dart';
import 'package:weather_cli/weather_exceptions.dart';
import 'package:weather_cli/weather_service.dart';

const _geocodeOk = '''
{
  "results": [
    {"name": "London", "country": "United Kingdom", "latitude": 51.5, "longitude": -0.12}
  ]
}
''';

const _forecastOk = '''
{
  "current": {
    "temperature_2m": 18.7,
    "relative_humidity_2m": 75,
    "wind_speed_10m": 12.2,
    "weather_code": 0
  }
}
''';

void main() {
  group('WeatherService', () {
    test('returns a WeatherReport for a known city', () async {
      final client = MockClient((request) async {
        if (request.url.host.contains('geocoding')) {
          return http.Response(_geocodeOk, 200);
        }
        return http.Response(_forecastOk, 200);
      });

      final service = WeatherService(client: client);
      final report = await service.getWeather('London');

      expect(report.city, 'London');
      expect(report.temperatureCelsius, 18.7);
      expect(report.description, 'Clear sky');
    });

    test('throws CityNotFoundException when geocoding has no results', () async {
      final client = MockClient((request) async {
        return http.Response('{"results": []}', 200);
      });

      final service = WeatherService(client: client);

      expect(
        () => service.getWeather('Nowhere'),
        throwsA(isA<CityNotFoundException>()),
      );
    });

    test('throws WeatherApiException on a non-200 response', () async {
      final client = MockClient((request) async {
        return http.Response('Server error', 500);
      });

      final service = WeatherService(client: client);

      expect(
        () => service.getWeather('London'),
        throwsA(isA<WeatherApiException>()),
      );
    });
  });
}
```

A companion `test/weather_report_test.dart` covers the pure JSON-parsing
logic directly — `describeWeatherCode`'s known and unknown codes, and
`WeatherReport.fromApi` with both well-formed and malformed input — the kind
of fast, dependency-free test that should make up the bulk of any suite.

```bash
dart test
```

```
00:00 +0: loading test/weather_report_test.dart
00:00 +4: All tests passed!
```

Real request/response shapes caught a genuine bug while building this
project: an early version of the `MockClient` checked
`request.url.path.contains('geocoding')`, but `geocoding-api.open-meteo.com`
puts "geocoding" in the **host**, not the path — so it silently matched the
wrong branch and every test call went through the "forecast" response. That
kind of mistake is exactly what a test written *before* trusting the
implementation is for.

## Stretch goals

- **Cache geocoding results.** City coordinates don't change — add a
  `Map<String, GeocodingResult>` cache inside `WeatherService` so repeated
  lookups of the same city skip the geocoding call entirely.
- **Add a `--units imperial` flag** that converts the temperature to
  Fahrenheit and wind speed to mph before printing, parsing it out of `args`
  before the city list.
- **Retry transient failures.** Wrap `_client.get` with a small retry loop
  (2-3 attempts with a short delay) for `WeatherApiException`, but *not* for
  `CityNotFoundException` — retrying won't fix a city that doesn't exist.
- **Add a `--json` output mode** that prints each `WeatherReport` as a JSON
  object (round-tripping the `toJson()` pattern from [Module
  5](05-json.md)) instead of the human-readable string, so the tool's
  output can be piped into another program.
