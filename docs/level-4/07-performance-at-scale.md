# 07 · Performance at Scale

[Performance & profiling](../level-3/08-performance-profiling.md) covered
measuring and fixing single-function hotspots. At scale, two different
techniques matter more: spreading CPU-bound work across multiple
[isolates](../level-3/05-isolates.md), and batching I/O so many small
operations don't each pay a fixed overhead cost individually.

## Parallelizing CPU work across isolates

A single isolate can only use one CPU core. Splitting a large computation
into chunks and running each on its own isolate lets independent work
actually run in parallel on a multi-core machine.

```dart
import 'dart:isolate';

bool isPrime(int n) {
  if (n < 2) return false;
  for (int i = 2; i * i <= n; i++) {
    if (n % i == 0) return false;
  }
  return true;
}

int countPrimesInRange(List<int> range) {
  int count = 0;
  for (int i = range[0]; i < range[1]; i++) {
    if (isPrime(i)) count++;
  }
  return count;
}

Future<void> main() async {
  const total = 2000000;
  const chunks = 4;
  final chunkSize = total ~/ chunks;

  final sw1 = Stopwatch()..start();
  final sequentialCount = countPrimesInRange([0, total]);
  sw1.stop();
  print('Sequential: $sequentialCount primes in ${sw1.elapsedMilliseconds}ms');

  final sw2 = Stopwatch()..start();
  final futures = <Future<int>>[];
  for (int i = 0; i < chunks; i++) {
    final start = i * chunkSize;
    final end = (i == chunks - 1) ? total : start + chunkSize;
    futures.add(Isolate.run(() => countPrimesInRange([start, end])));
  }
  final results = await Future.wait(futures);
  final parallelCount = results.reduce((a, b) => a + b);
  sw2.stop();
  print('Parallel ($chunks isolates): $parallelCount primes in ${sw2.elapsedMilliseconds}ms');
}
// Sequential: 148933 primes in 213ms
// Parallel (4 isolates): 148933 primes in 83ms
```

A ~2.5x speedup from 4 isolates on this machine, not a full 4x — spawning
isolates and copying the chunk boundaries/results across isolate
boundaries (see [module 05](../level-3/05-isolates.md)) has its own cost,
and that overhead eats into the theoretical maximum. Chunking too finely
(many tiny isolates) can make that overhead dominate and net out *slower*
than staying sequential — always measure the actual speedup for your
workload's size rather than assuming isolates are free parallelism.

## Batching database writes

[Databases](../level-3/03-databases.md) showed `db.prepare` for
parameterized inserts. At any real volume, wrapping many writes in a single
transaction is the difference between each write paying disk-sync overhead
individually versus once for the whole batch.

```dart
import 'package:sqlite3/sqlite3.dart';

void main() {
  const n = 5000;

  final db1 = sqlite3.open('/tmp/individual.db');
  db1.execute('CREATE TABLE items (id INTEGER PRIMARY KEY, value INTEGER);');
  final sw1 = Stopwatch()..start();
  for (int i = 0; i < n; i++) {
    db1.execute('INSERT INTO items (value) VALUES ($i)'); // its own implicit transaction
  }
  sw1.stop();
  print('$n individual inserts: ${sw1.elapsedMilliseconds}ms');
  db1.dispose();

  final db2 = sqlite3.open('/tmp/batched.db');
  db2.execute('CREATE TABLE items (id INTEGER PRIMARY KEY, value INTEGER);');
  final sw2 = Stopwatch()..start();
  db2.execute('BEGIN TRANSACTION');
  final stmt = db2.prepare('INSERT INTO items (value) VALUES (?)');
  for (int i = 0; i < n; i++) {
    stmt.execute([i]);
  }
  stmt.dispose();
  db2.execute('COMMIT');
  sw2.stop();
  print('$n batched inserts in one transaction: ${sw2.elapsedMilliseconds}ms');
  db2.dispose();
}
// 5000 individual inserts: 2257ms
// 5000 batched inserts in one transaction: 8ms
```

That's roughly a 280x difference on a real on-disk database (an in-memory
database shows far less of a gap, since there's no disk sync cost to
amortize in the first place — always benchmark against the storage you
actually deploy on). Every statement outside an explicit transaction runs
in its own implicit one, and each commit involves a disk sync; wrapping
5,000 writes in one `BEGIN`/`COMMIT` pays that sync cost once instead of
5,000 times.

## The trap: over-batching risks losing more work on failure

Batching isn't free of tradeoffs — a batch that fails partway through (a
constraint violation on row 4,999 of 5,000) rolls back the **entire**
transaction by default, discarding rows 1 through 4,998 that were
individually valid. The fix is choosing a batch size that balances
throughput against acceptable retry cost: commit every few hundred/thousand
rows instead of one all-or-nothing transaction, so a failure only loses
one batch's worth of work, not the whole run.

## Cheat sheet

| Technique | When to reach for it |
|---|---|
| `Isolate.run` per chunk | CPU-bound work large enough that isolate overhead is worth it |
| Measure actual speedup | Isolate overhead can make fine-grained chunking net slower |
| `BEGIN`/`COMMIT` around many writes | Batches disk-sync cost across many operations instead of paying it per-row |
| Implicit per-statement transactions | The default without an explicit `BEGIN` — one sync per write |
| Bounded batch size | Balances throughput against how much work one failure can roll back |
| Benchmark on real storage | In-memory vs. on-disk DBs show very different batching payoffs |

## Exercise

Extend the prime-counting example to accept a `chunks` parameter and try
1, 2, 4, and 8 chunks against the same `total`, printing elapsed time for
each and noting where adding more chunks stops helping (or starts hurting)
on your machine. Separately, modify the batched-insert example to commit
every 500 rows instead of one transaction for all 5,000, and time it against
both the fully-individual and fully-batched versions — confirm it lands
between the two.
