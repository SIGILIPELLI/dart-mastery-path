# 03 · Control Flow

## `if` / `else if` / `else`

```dart
void main() {
  int score = 82;

  if (score >= 90) {
    print('A');
  } else if (score >= 80) {
    print('B');
  } else if (score >= 70) {
    print('C');
  } else {
    print('F');
  }
  // B
}
```

Dart requires the condition to be a `bool` — there's no implicit truthiness
for numbers or strings, unlike JavaScript or Python.

```dart
// if (0) { ... }        // compile error: int is not bool
// if ('') { ... }        // compile error: String is not bool
```

## `switch` statements and expressions

Dart's `switch` supports classic `case` statements with fallthrough control,
plus (since Dart 3) a more concise `switch` *expression* form.

```dart
void main() {
  String day = 'TUE';

  // Classic switch statement
  switch (day) {
    case 'MON':
    case 'TUE':
    case 'WED':
    case 'THU':
    case 'FRI':
      print('Weekday');
      break;
    case 'SAT':
    case 'SUN':
      print('Weekend');
      break;
    default:
      print('Unknown');
  }
  // Weekday
}
```

```dart
void main() {
  int month = 4;

  // Switch expression -- evaluates to a value directly, no break needed
  String season = switch (month) {
    12 || 1 || 2 => 'Winter',
    3 || 4 || 5 => 'Spring',
    6 || 7 || 8 => 'Summer',
    9 || 10 || 11 => 'Fall',
    _ => 'Invalid month',
  };

  print(season);   // Spring
}
```

The switch expression form is exhaustiveness-checked by the compiler and is
generally preferred in modern Dart when you're producing a value.

## `for` loops

```dart
void main() {
  // Classic C-style for loop
  for (int i = 0; i < 3; i++) {
    print('i = $i');
  }
  // i = 0
  // i = 1
  // i = 2

  // for-in loop over an iterable
  var fruits = ['apple', 'banana', 'cherry'];
  for (var fruit in fruits) {
    print(fruit);
  }
  // apple
  // banana
  // cherry
}
```

## `while` and `do-while`

```dart
void main() {
  int countdown = 3;
  while (countdown > 0) {
    print(countdown);
    countdown--;
  }
  print('Liftoff!');
  // 3
  // 2
  // 1
  // Liftoff!

  int n = 0;
  do {
    print('runs at least once, n=$n');
    n++;
  } while (n < 0);
  // runs at least once, n=0
}
```

## `break` and `continue`

```dart
void main() {
  for (int i = 0; i < 10; i++) {
    if (i == 5) break;          // exit the loop entirely
    if (i.isOdd) continue;      // skip the rest of this iteration
    print(i);
  }
  // 0
  // 2
  // 4
}
```

## Labeled loops

Dart supports labels so `break`/`continue` can target an outer loop from
inside a nested one:

```dart
void main() {
  outer:
  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      if (j == 1) continue outer;
      print('i=$i j=$j');
    }
  }
  // i=0 j=0
  // i=1 j=0
  // i=2 j=0
}
```

## Ternary and null-aware operators

```dart
void main() {
  int age = 20;
  String category = age >= 18 ? 'adult' : 'minor';
  print(category);   // adult

  String? nickname;
  String displayName = nickname ?? 'Anonymous';   // ?? -- use right side if left is null
  print(displayName);   // Anonymous

  nickname ??= 'Ada';   // assign only if currently null
  print(nickname);      // Ada
}
```

## Cheat sheet

| Construct | Use when |
|-----------|----------|
| `if`/`else if`/`else` | Branching on arbitrary boolean conditions |
| `switch` statement | Branching on discrete values with side effects per case |
| `switch` expression | Producing a value from discrete cases, exhaustively checked |
| `for` | Known number of iterations, or iterating a collection |
| `while` | Unknown number of iterations, condition checked before body |
| `do-while` | Same, but body always runs at least once |
| `break` / `continue` | Exit a loop early / skip to the next iteration |
| `?:` | Inline conditional expression |
| `??` / `??=` | Fallback for `null`, or assign-if-null |

## Exercise

Write a program that loops from 1 to 30 and, for each number, prints "Fizz"
if divisible by 3, "Buzz" if divisible by 5, "FizzBuzz" if divisible by both,
and the number itself otherwise. Then write a `switch` expression that maps an
`int` 1-7 to its weekday name, and print the result for a couple of test
values including an out-of-range one (using `_` as the default case).
