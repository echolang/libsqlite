# libsqlite

SQLite for Echo, on the system's libsqlite3. A connection, prepared statements, typed binds and column reads. Nothing more.

- `exec` when you want SQL to do something. `prepare` when you want rows back or values bound.
- Binds are typed: `.int`, `.float`, `.text`, `.blob`, `.none`. Positional, from 0.
- Column reads return null for NULL. `kind` tells you what SQLite stored.
- Failures are values. An `Error` keeps SQLite's extended result code, so a UNIQUE violation is not just "constraint failed".
- Transactions, pragmas and schema are plain SQL through `exec`. No wrapper to learn.
- A connection is serialized: handing it to another thread is safe. Statements stay on the thread that prepared them. For parallel work, open one connection per thread.

## Installation

From your project directory:

```bash
epm add echolang/libsqlite --git https://github.com/echolang/libsqlite --range ^0.1
```

That writes a `#[requires:]` line and vendors the sources. Echo sees the `sqlite` namespace as soon as the module loads. The module links `-lsqlite3`, so libsqlite3 (3.37 or newer) has to be on the machine:

- **macOS**: nothing to do, it ships with the system.
- **Debian/Ubuntu**: `sudo apt-get install libsqlite3-dev`
- **Windows**: `vcpkg install sqlite3:x64-windows-static`, then point the linker at it: `echoc test --link "search:<vcpkg>\installed\x64-windows-static\lib"`.

## Quick start

Open, `exec`, insert, read:

```echo
sqlite::Database $db = guard sqlite::Database::open('notes.sqlite') else ($e) {
    die("open: {$e}");
}

guard $db->exec('CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, body TEXT NOT NULL)') else ($e) {
    die("schema: {$e}");
}

sqlite::Statement $ins = guard $db->prepare('INSERT INTO notes (body) VALUES (?)') else ($e) {
    die("prepare: {$e}");
}
guard $ins->bind([.text('hello')]) else ($e) { die("bind: {$e}"); }
guard $ins->run() else ($e) { die("insert: {$e}"); }
int64 $id = $db->lastInsertId();
echo "inserted {$id}";

sqlite::Statement $q = guard $db->prepare('SELECT id, body FROM notes ORDER BY id') else ($e) {
    die("prepare: {$e}");
}
while (true) {
    bool $row = guard $q->step() else ($e) { die("select: {$e}"); }
    if (!$row) {
        break;
    }
    int64 $n = $q->int(0) ?? 0;
    string $body = $q->text(1) ?? '';
    echo "{$n}: {$body}";
}
```

`sqlite::Database::memory()` gives you a private in-memory database instead, handy for tests.

## Opening

`open` takes a path and a `Mode`:

| Mode | Does |
| --- | --- |
| `.create` (default) | read and write, create the file if it is missing |
| `.readWrite` | read and write, the file has to exist |
| `.readOnly` | read only, the file has to exist |

```echo
sqlite::Database $ro = guard sqlite::Database::open('notes.sqlite', mode: .readOnly) else ($e) {
    die("open: {$e}");
}
```

`:memory:` and `file:` URIs work as paths too. `busyTimeout` sets how long a statement waits for another connection's lock before it fails as busy:

```echo
$db->busyTimeout(time::duration::secs(5));
```

The connection closes when the last holder lets go. `close()` closes it now. A statement keeps its connection alive, so statements still held keep the file open until they are dropped, but once `close()` ran they refuse to bind or step. Don't call `close()` while another thread is using the connection.

## Statements and values

A statement is compiled once and runs as often as you like. `run()` steps it to the end and resets it, so you can bind and run again straight away:

```echo
sqlite::Statement $ins = guard $db->prepare('INSERT INTO t (i, f, s, b, n) VALUES (?, ?, ?, ?, ?)');
guard $ins->bind([.int(42), .float(1.25), .text('grüezi'), .blob($bytes), .none]);
guard $ins->run();

guard $ins->bind(0, .int(43));
guard $ins->run();
```

`bind(array)` sets every parameter in order. `bind($at, $value)` sets one. A statement you stepped by hand has to be `reset()` before it takes new bindings; `reset(true)` also drops the old ones.

On a row, read columns by position: `int`, `float`, `text`, `blob`. Each returns null when the column is NULL. `isNull`, `kind` and `name` tell you about a column, `columns()` how many there are. Text and blobs are copied when bound and when read, so nothing points into SQLite's memory.

After a write, `changes()` is the number of rows it touched and `lastInsertId()` the rowid it created.

## Transactions

Plain SQL:

```echo
guard $db->exec('BEGIN IMMEDIATE');
// ...
guard $db->exec('COMMIT');
```

`autocommit()` is false while a transaction is open. Pragmas work the same way: `$db->exec('PRAGMA journal_mode = WAL')`.

## Errors

Every call that can fail returns `status<Error>` or `result<T, Error>`. `"{$e}"` reads like `sqlite error 2067: UNIQUE constraint failed: notes.body`. `$e->code` is the extended result code and `$e->message` SQLite's text, and there are helpers for the codes you usually branch on:

```echo
function addNote(sqlite::Statement $ins) : void
{
    guard $ins->run() else ($e) {
        if ($e->unique()) {
            echo "already there";
            return;
        }
        if ($e->busy()) {
            echo "locked, try again";
            return;
        }
        die("{$e}");
    }
    echo "added";
}
```

`unique()`, `foreignKey()`, `notNull()`, `busy()`, and `is(Code)` / `primary()` for the rest.

## What isn't here

- No connection pool. Open one connection per thread.
- No named parameters (`:name`), only positional.
- No backup, blob streaming, user functions or hooks.
- No bundled SQLite: it is the system's, 3.37 or newer. `sqlite::Database::version()` tells you which.
