---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Dart Mastery Path with a 18-question final quiz."
---

# Final Quiz

18 questions drawn from all four levels of this course. You need **70% (13 out of 18)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "In Dart, what does `String name` mean under sound null safety?",
    "options": [
      "name can never be null; use String? to allow null",
      "name may be null",
      "name is optional",
      "name is dynamic"
    ],
    "correctIndex": 0,
    "explanation": "Non-nullable by default."
  },
  {
    "question": "What does this print?\n```dart\nvoid main(){ String? s; print(s?.length); print(s ?? 'none'); s ??= 'x'; print(s); }\n```\n",
    "options": [
      "null\nnull\nx",
      "Error",
      "0\nnone\nx",
      "null\nnone\nx"
    ],
    "correctIndex": 3,
    "explanation": "?. short-circuits, ?? gives a default, ??= assigns if null. Verified by running it with the Dart SDK."
  },
  {
    "question": "What does this print?\n```dart\nvoid main(){ final a=[1,2]; a.add(3); print(a); const b=[1,2]; print(b.length); }\n```\n",
    "options": [
      "[1, 2]\n2",
      "Error",
      "[1, 2, 3]\n2",
      "[1, 2, 3]\n3"
    ],
    "correctIndex": 2,
    "explanation": "final stops reassignment, not mutation; const makes an immutable compile-time value. Verified by running it with the Dart SDK."
  },
  {
    "question": "What does this print?\n```dart\nvoid main(){ var n=[1,2,3,4]; print(n.where((x)=>x.isOdd).map((x)=>x*10).toList()); }\n```\n",
    "options": [
      "[1, 3]",
      "[10, 20, 30, 40]",
      "[10, 30]",
      "Error"
    ],
    "correctIndex": 2,
    "explanation": "where filters, map transforms. Verified by running it with the Dart SDK."
  },
  {
    "question": "What does this print?\n```dart\nclass A{ A(){ print('A'); } }\nclass B extends A{ B(){ print('B'); } }\nvoid main(){ B(); }\n```\n",
    "options": [
      "A",
      "B",
      "B\nA",
      "A\nB"
    ],
    "correctIndex": 3,
    "explanation": "The superclass constructor runs first. Verified by running it with the Dart SDK."
  },
  {
    "question": "What does this print?\n```dart\nmixin Walker{ String move()=>'walk'; }\nclass Dog with Walker{}\nvoid main(){ print(Dog().move()); }\n```\n",
    "options": [
      "Error",
      "Dog",
      "walk",
      "move"
    ],
    "correctIndex": 2,
    "explanation": "Mixins add reusable behavior. Verified by running it with the Dart SDK."
  },
  {
    "question": "What does this print?\n```dart\nvoid main() async { print('a'); await Future.delayed(Duration(milliseconds:5)); print('b'); }\n```\n",
    "options": [
      "a",
      "b\na",
      "Error",
      "a\nb"
    ],
    "correctIndex": 3,
    "explanation": "await pauses this function only. Verified by running it with the Dart SDK."
  },
  {
    "question": "What does this print?\n```dart\nStream<int> s() async* { for (var i=1;i<=3;i++) yield i*i; }\nvoid main() async { await for (final v in s()) print(v); }\n```\n",
    "options": [
      "Error",
      "1\n4\n9",
      "9",
      "1\n2\n3"
    ],
    "correctIndex": 1,
    "explanation": "async* generators emit with yield. Verified by running it with the Dart SDK."
  },
  {
    "question": "What does this print?\n```dart\nimport 'dart:isolate';\nvoid main() async { print(await Isolate.run(()=>List.generate(5,(i)=>i*i).reduce((a,b)=>a+b))); }\n```\n",
    "options": [
      "Error",
      "5",
      "30",
      "10"
    ],
    "correctIndex": 2,
    "explanation": "CPU-heavy work can run in a separate isolate. Verified by running it with the Dart SDK."
  },
  {
    "question": "Why isn't async/await the same as parallelism in Dart?",
    "options": [
      "async uses threads",
      "They're identical",
      "isolates share memory",
      "It interleaves work on one thread via the event loop; isolates give real parallelism with separate memory"
    ],
    "correctIndex": 3,
    "explanation": "Use isolates for CPU-bound work."
  },
  {
    "question": "A Flutter list janks because build() sorts 10,000 items. What's the fix?",
    "options": [
      "Call setState more",
      "Move the work out of build() (compute once or in an isolate) and use const widgets where possible",
      "Use sleep",
      "Ignore"
    ],
    "correctIndex": 1,
    "explanation": "build() must be cheap."
  },
  {
    "question": "State is passed through six widgets only to be used at the bottom. What do you use?",
    "options": [
      "Globals",
      "A state management approach (e.g. Provider/Riverpod/Bloc) to lift and share state",
      "More constructors",
      "setState everywhere"
    ],
    "correctIndex": 1,
    "explanation": "Reduces prop drilling."
  },
  {
    "question": "A shelf handler needs logging and auth for every route. How?",
    "options": [
      "Copy code",
      "A mixin",
      "Wrap handlers with middleware in a Pipeline",
      "Use globals"
    ],
    "correctIndex": 2,
    "explanation": "Middleware composes behavior."
  },
  {
    "question": "A query builds SQL by concatenating a request parameter. What's wrong?",
    "options": [
      "Style",
      "Slow",
      "SQL injection risk: use parameterized queries with placeholders",
      "Nothing"
    ],
    "correctIndex": 2,
    "explanation": "Bind values."
  },
  {
    "question": "Where must an API secret not be stored in a Flutter app?",
    "options": [
      "In vault",
      "In the client bundle, since it can be extracted; keep it on a backend",
      "On server",
      "In env on server"
    ],
    "correctIndex": 1,
    "explanation": "Clients are untrusted."
  },
  {
    "question": "What's the smallest reasonable Docker setup for a Dart server?",
    "options": [
      "Copy source",
      "Ship the SDK",
      "Compile with `dart compile exe` in a build stage and copy the binary into a minimal runtime image",
      "Run dart run"
    ],
    "correctIndex": 2,
    "explanation": "Small and fast to start."
  },
  {
    "question": "A package change breaks callers' code. What version bump?",
    "options": [
      "Major, per semantic versioning",
      "Patch",
      "None",
      "Minor"
    ],
    "correctIndex": 0,
    "explanation": "Breaking changes require a major."
  },
  {
    "question": "CI for a Dart repo should at minimum run...",
    "options": [
      "Only build",
      "dart analyze, dart format --set-exit-if-changed and dart test",
      "Nothing",
      "Manual"
    ],
    "correctIndex": 1,
    "explanation": "Gate merges automatically."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Totals and counts with collections

Implement total(items) with fold (sum of qty x price) and countByInitial(items) returning the total quantity per upper-cased first letter of the item name. Print the total to two decimals and the map.

**Starter code**

```dart
class Item {
  final String name;
  final int qty;
  final double price;
  Item(this.name, this.qty, this.price);
}

double total(List<Item> items) {
  // sum qty * price with fold
  return 0;
}

Map<String, int> countByInitial(List<Item> items) {
  // total quantity per upper-cased first letter of the name
  return {};
}

void main() {
  final items = [
    Item('apple', 3, 0.5),
    Item('avocado', 2, 1.25),
    Item('bread', 1, 2.0),
  ];
  print(total(items).toStringAsFixed(2));
  print(countByInitial(items));
}
```

**Expected output**

```text
6.00
{A: 5, B: 1}
```

<details><summary>Show solution and explanation</summary>

<pre><code>class Item {
  final String name;
  final int qty;
  final double price;
  Item(this.name, this.qty, this.price);
}

double total(List&lt;Item&gt; items) =&gt;
    items.fold(0.0, (sum, i) =&gt; sum + i.qty * i.price);

Map&lt;String, int&gt; countByInitial(List&lt;Item&gt; items) {
  final counts = &lt;String, int&gt;{};
  for (final i in items) {
    final key = i.name[0].toUpperCase();
    counts[key] = (counts[key] ?? 0) + i.qty;
  }
  return counts;
}

void main() {
  final items = [
    Item(&#x27;apple&#x27;, 3, 0.5),
    Item(&#x27;avocado&#x27;, 2, 1.25),
    Item(&#x27;bread&#x27;, 1, 2.0),
  ];
  print(total(items).toStringAsFixed(2));
  print(countByInitial(items));
}
</code></pre>

<p>fold accumulates a running sum, and a map with `?? 0` accumulates per-key totals. Dart prints maps as {A: 5, B: 1}. Verified by running the solution with the Dart SDK.</p>
</details>

### Typed exceptions and streams

Complete Account.withdraw (ArgumentError for non-positive amounts, a custom InsufficientFunds exception when over balance), and an async* generator ticks(n) that yields 1..n. The provided main handles each failure type and uses finally.

**Starter code**

```dart
import 'dart:async';

class Account {
  int _balance;
  Account(this._balance);
  int get balance => _balance;
  void withdraw(int amount) {
    // throw ArgumentError for amount <= 0, InsufficientFunds(shortBy) when amount > balance,
    // otherwise subtract it from the balance
  }
}

class InsufficientFunds implements Exception {
  final int shortBy;
  InsufficientFunds(this.shortBy);
  @override
  String toString() => 'InsufficientFunds(short by $shortBy)';
}

Stream<int> ticks(int n) async* {
  // yield 1..n
}

Future<void> main() async {
  final acct = Account(100);
  for (final amt in [30, 100, -5]) {
    try {
      acct.withdraw(amt);
      print('withdrew $amt, balance ${acct.balance}');
    } on InsufficientFunds catch (e) {
      print('error: $e');
    } on ArgumentError catch (e) {
      print('invalid: ${e.message}');
    } finally {
      print('checked $amt');
    }
  }
  final total = await ticks(4).fold<int>(0, (a, b) => a + b);
  print('sum of ticks: $total');
}
```

**Expected output**

```text
withdrew 30, balance 70
checked 30
error: InsufficientFunds(short by 30)
checked 100
invalid: amount must be positive
checked -5
sum of ticks: 10
```

<details><summary>Show solution and explanation</summary>

<pre><code>import &#x27;dart:async&#x27;;

class Account {
  int _balance;
  Account(this._balance);
  int get balance =&gt; _balance;
  void withdraw(int amount) {
    if (amount &lt;= 0) throw ArgumentError(&#x27;amount must be positive&#x27;);
    if (amount &gt; _balance) throw InsufficientFunds(amount - _balance);
    _balance -= amount;
  }
}

class InsufficientFunds implements Exception {
  final int shortBy;
  InsufficientFunds(this.shortBy);
  @override
  String toString() =&gt; &#x27;InsufficientFunds(short by $shortBy)&#x27;;
}

Stream&lt;int&gt; ticks(int n) async* {
  for (var i = 1; i &lt;= n; i++) {
    yield i;
  }
}

Future&lt;void&gt; main() async {
  final acct = Account(100);
  for (final amt in [30, 100, -5]) {
    try {
      acct.withdraw(amt);
      print(&#x27;withdrew $amt, balance ${acct.balance}&#x27;);
    } on InsufficientFunds catch (e) {
      print(&#x27;error: $e&#x27;);
    } on ArgumentError catch (e) {
      print(&#x27;invalid: ${e.message}&#x27;);
    } finally {
      print(&#x27;checked $amt&#x27;);
    }
  }
  final total = await ticks(4).fold&lt;int&gt;(0, (a, b) =&gt; a + b);
  print(&#x27;sum of ticks: $total&#x27;);
}
</code></pre>

<p>Catch the most specific exception types first, use finally for always-run code, and async* with yield to produce a Stream. Verified by running the solution with the Dart SDK.</p>
</details>
