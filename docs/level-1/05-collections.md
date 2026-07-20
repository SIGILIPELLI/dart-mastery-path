# 05 · Collections (List, Map, Set)

## Lists

A `List` is an ordered, indexable collection — Dart's equivalent of an array,
but growable by default.

```dart
void main() {
  var fruits = ['apple', 'banana', 'cherry'];

  print(fruits[0]);          // apple
  print(fruits.length);      // 3

  fruits.add('date');
  fruits.insert(1, 'apricot');
  print(fruits);
  // [apple, apricot, banana, cherry, date]

  fruits.remove('banana');
  fruits.removeAt(0);
  print(fruits);
  // [apricot, cherry, date]

  print(fruits.contains('cherry'));   // true
  print(fruits.indexOf('date'));      // 2
}
```

Fixed-length lists (can't add/remove, but elements are still mutable) and
truly immutable lists are both available:

```dart
void main() {
  var fixed = List.filled(3, 0);   // [0, 0, 0], fixed-length
  fixed[0] = 99;                    // OK -- elements are mutable
  // fixed.add(1);                  // runtime error -- can't grow a fixed-length list

  var immutable = const [1, 2, 3];
  // immutable.add(4);              // compile error -- can't modify a const list
  print(immutable);   // [1, 2, 3]
}
```

## Maps

A `Map` stores key-value pairs, similar to a dictionary or hash map.

```dart
void main() {
  var ages = {'Ada': 36, 'Grace': 85, 'Linus': 55};

  print(ages['Ada']);          // 36
  print(ages['Unknown']);      // null -- missing keys return null, not an error

  ages['Ada'] = 37;             // update
  ages['Katherine'] = 101;      // insert
  print(ages);
  // {Ada: 37, Grace: 85, Linus: 55, Katherine: 101}

  ages.remove('Linus');
  print(ages.containsKey('Grace'));   // true
  print(ages.keys);      // (Ada, Grace, Katherine)
  print(ages.values);    // (37, 85, 101)

  ages.forEach((name, age) {
    print('$name is $age');
  });
}
```

`Map.entries` and destructuring give a convenient way to iterate both key and
value together:

```dart
void main() {
  var ages = {'Ada': 36, 'Grace': 85};
  for (var entry in ages.entries) {
    print('${entry.key}: ${entry.value}');
  }
  // Ada: 36
  // Grace: 85
}
```

## Sets

A `Set` is an unordered collection of unique elements — duplicates are
automatically discarded.

```dart
void main() {
  var uniqueNumbers = {1, 2, 2, 3, 3, 3};
  print(uniqueNumbers);   // {1, 2, 3}

  uniqueNumbers.add(4);
  uniqueNumbers.add(2);    // no-op, already present
  print(uniqueNumbers);   // {1, 2, 3, 4}

  var a = {1, 2, 3};
  var b = {2, 3, 4};
  print(a.union(b));           // {1, 2, 3, 4}
  print(a.intersection(b));    // {2, 3}
  print(a.difference(b));      // {1}
}
```

Note: `{}` alone creates an empty `Map`, not a `Set` — for an empty set, use
`<int>{}` or `Set<int>()`.

## Spread operator and collection-if / collection-for

```dart
void main() {
  var veggies = ['carrot', 'potato'];
  var fruits = ['apple', 'banana'];

  // Spread -- inline one collection's elements into another
  var groceries = [...fruits, ...veggies, 'bread'];
  print(groceries);
  // [apple, banana, carrot, potato, bread]

  bool includeDessert = true;
  var menu = [
    'soup',
    'salad',
    if (includeDessert) 'cake',   // collection-if -- conditionally include an element
  ];
  print(menu);   // [soup, salad, cake]

  var doubled = [for (var v in [1, 2, 3]) v * 2];   // collection-for
  print(doubled);   // [2, 4, 6]
}
```

## Iteration and common methods

```dart
void main() {
  var numbers = [5, 3, 8, 1, 9, 2];

  numbers.sort();
  print(numbers);   // [1, 2, 3, 5, 8, 9]

  print(numbers.first);       // 1
  print(numbers.last);        // 9
  print(numbers.reversed.toList());   // [9, 8, 5, 3, 2, 1]

  var evens = numbers.where((n) => n.isEven).toList();
  print(evens);   // [2, 8]

  var doubled = numbers.map((n) => n * 2).toList();
  print(doubled);   // [2, 4, 6, 10, 16, 18]

  int sum = numbers.fold(0, (total, n) => total + n);
  print(sum);   // 28
}
```

## Cheat sheet

| Type | Literal | Ordered? | Duplicates? | Access |
|------|---------|----------|-------------|--------|
| `List<T>` | `[1, 2, 3]` | Yes | Yes | By index: `list[0]` |
| `Map<K, V>` | `{'a': 1}` | Insertion order | Keys unique | By key: `map['a']` |
| `Set<T>` | `{1, 2, 3}` | Insertion order | No | By value: `set.contains(x)` |

## Exercise

Given a `List<Map<String, dynamic>>` where each map represents a person with
`'name'` (String) and `'age'` (int) keys, write code that: filters to only
people 18 or older, extracts just their names into a new `List<String>`,
sorts that list alphabetically, and prints the result. Then build a `Set<int>`
from two overlapping lists of numbers and print their union, intersection, and
difference.
