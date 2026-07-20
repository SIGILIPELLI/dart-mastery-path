# 10 · Project — CLI To-Do App

Time to combine everything from this level — variables, control flow,
functions, collections, classes, null safety, and a touch of async — into one
working command-line program: a persistent to-do list.

## What we're building

A CLI tool that lets you:

- add a task
- list all tasks (with a done/pending marker)
- mark a task as done
- remove a task
- persist tasks to a local JSON file between runs

## Project setup

```bash
dart create todo_cli
cd todo_cli
```

We'll keep everything in `bin/todo_cli.dart` for simplicity — real projects
would split the `Task` model and storage logic into `lib/`, as shown in
[Module 9](09-packages.md).

## Modeling a `Task`

```dart
class Task {
  final String title;
  bool done;

  Task(this.title, {this.done = false});

  // Convert this Task to a simple Map so it can be JSON-encoded
  Map<String, dynamic> toJson() => {
        'title': title,
        'done': done,
      };

  // Rebuild a Task from a decoded JSON Map
  factory Task.fromJson(Map<String, dynamic> json) {
    return Task(json['title'] as String, done: json['done'] as bool);
  }

  @override
  String toString() {
    final marker = done ? '[x]' : '[ ]';
    return '$marker $title';
  }
}
```

`factory` constructors don't always create a new instance directly — here it
just delegates to the normal constructor after unpacking the JSON map, but the
`factory` keyword also allows returning a cached instance or a subtype, which
a plain constructor can't do.

## Reading and writing the task list as JSON

```dart
import 'dart:convert';
import 'dart:io';

const _storageFile = 'tasks.json';

List<Task> loadTasks() {
  final file = File(_storageFile);
  if (!file.existsSync()) return [];

  final contents = file.readAsStringSync();
  if (contents.trim().isEmpty) return [];

  final List<dynamic> decoded = jsonDecode(contents);
  return decoded.map((item) => Task.fromJson(item as Map<String, dynamic>)).toList();
}

void saveTasks(List<Task> tasks) {
  final file = File(_storageFile);
  final encoded = jsonEncode(tasks.map((t) => t.toJson()).toList());
  file.writeAsStringSync(encoded);
}
```

## Command handling

```dart
void printUsage() {
  print('''
Usage: dart run bin/todo_cli.dart <command> [args]

Commands:
  add <title>       Add a new task
  list               List all tasks
  done <index>       Mark task at <index> as done
  remove <index>     Remove task at <index>
''');
}

void main(List<String> args) {
  final tasks = loadTasks();

  if (args.isEmpty) {
    printUsage();
    return;
  }

  final command = args[0];

  switch (command) {
    case 'add':
      if (args.length < 2) {
        print('Error: "add" requires a title.');
        return;
      }
      final title = args.sublist(1).join(' ');
      tasks.add(Task(title));
      saveTasks(tasks);
      print('Added: $title');

    case 'list':
      if (tasks.isEmpty) {
        print('No tasks yet. Add one with: add <title>');
        return;
      }
      for (var i = 0; i < tasks.length; i++) {
        print('$i. ${tasks[i]}');
      }

    case 'done':
      final index = _parseIndex(args, tasks.length);
      if (index == null) return;
      tasks[index].done = true;
      saveTasks(tasks);
      print('Marked done: ${tasks[index].title}');

    case 'remove':
      final index = _parseIndex(args, tasks.length);
      if (index == null) return;
      final removed = tasks.removeAt(index);
      saveTasks(tasks);
      print('Removed: ${removed.title}');

    default:
      print('Unknown command: $command');
      printUsage();
  }
}

int? _parseIndex(List<String> args, int taskCount) {
  if (args.length < 2) {
    print('Error: this command requires an index.');
    return null;
  }
  final index = int.tryParse(args[1]);
  if (index == null || index < 0 || index >= taskCount) {
    print('Error: invalid index "${args[1]}".');
    return null;
  }
  return index;
}
```

## Running it

```bash
dart run bin/todo_cli.dart add "Buy groceries"
# Added: Buy groceries

dart run bin/todo_cli.dart add "Write Dart lesson"
# Added: Write Dart lesson

dart run bin/todo_cli.dart list
# 0. [ ] Buy groceries
# 1. [ ] Write Dart lesson

dart run bin/todo_cli.dart done 0
# Marked done: Buy groceries

dart run bin/todo_cli.dart list
# 0. [x] Buy groceries
# 1. [ ] Write Dart lesson

dart run bin/todo_cli.dart remove 1
# Removed: Write Dart lesson
```

Tasks persist in `tasks.json` next to wherever you run the command from, so
running `list` again in a new terminal session still shows your saved tasks.

## Compiling to a standalone binary

```bash
dart compile exe bin/todo_cli.dart -o todo
./todo list
```

## What you practiced

| Level 1 topic | Where it showed up |
|----------------|---------------------|
| Variables & types | `title`, `done`, `_storageFile` |
| Control flow | `switch` on `command`, `if` guards |
| Functions | `printUsage`, `_parseIndex`, `loadTasks`, `saveTasks` |
| Collections | `List<Task>`, iterating and indexing |
| Classes | The `Task` model, constructors, `toJson`/`fromJson` |
| Null safety | `int? _parseIndex(...)`, `tryParse`, null checks |
| Packages | `dart:convert` and `dart:io` core libraries |

## Exercise

Extend the to-do app with a `due <index> <YYYY-MM-DD>` command that attaches
an optional `DateTime? dueDate` to a task, and update `list` to show overdue
tasks (due date before `DateTime.now()` and not yet done) with an `[OVERDUE]`
marker. You'll need to extend `Task.toJson`/`Task.fromJson` to
serialize/deserialize the date as an ISO 8601 string (`DateTime.parse` /
`.toIso8601String()`).
