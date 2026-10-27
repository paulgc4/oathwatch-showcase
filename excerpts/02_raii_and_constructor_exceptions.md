# Excerpt: RAII, and a constructor that throws

How the SQLite connection is owned. This is the fix for a leak that AddressSanitizer found. From `state_store.hpp` and `state_store.cpp` (excerpts only).

```cpp
// state_store.hpp
struct SqliteCloser {
  void operator()(sqlite3* db) const;
};
...
 private:
  std::unique_ptr<sqlite3, SqliteCloser> db_;   // was: sqlite3* db_
```

```cpp
// state_store.cpp
StateStore::StateStore(const std::string& path, int busy_timeout_ms) {
#ifndef _WIN32
  // The database records which hardening checks fail, which is useful to an
  // attacker. Create it owner-only; SQLite gives -wal/-shm the same mode.
  if (path != ":memory:" && !path.empty()) {
    int fd = ::open(path.c_str(), O_RDWR | O_CREAT | O_CLOEXEC, 0600);
    if (fd >= 0) ::close(fd);
  }
#endif
  sqlite3* raw = nullptr;
  int rc = sqlite3_open(path.c_str(), &raw);
  db_.reset(raw);  // owned from here on: closed even if the rest of this constructor throws
  if (rc != SQLITE_OK) {
    std::string msg = db_ ? sqlite3_errmsg(db_.get()) : "out of memory";
    throw std::runtime_error("cannot open database '" + path + "': " + msg);
  }
  // Must come before the first statement: switching to WAL and creating the
  // schema both need locks, and another process (a second supervisor, an
  // operator's sqlite3 shell, a monitoring script) may be opening the same file.
  sqlite3_busy_timeout(db_.get(), busy_timeout_ms);
  exec_init("PRAGMA journal_mode=WAL;");
  exec_init(kSchema);
}

StateStore::~StateStore() = default;

void SqliteCloser::operator()(sqlite3* db) const { sqlite3_close(db); }
```

**The bug.** The first version held a raw `sqlite3*`. If the constructor threw after `sqlite3_open` succeeded (a corrupt database file, say), the object was never fully constructed, so its destructor never ran and the connection leaked.

**The fix.** When a constructor throws, the half-built object's destructor does not run, but every member that finished constructing is destroyed. Putting the handle in a `unique_ptr` with a custom deleter closes it during unwinding. `db_.reset(raw)` is where ownership begins.

**Also in this function:** the file is created 0600, the busy timeout is set before the first statement, and setup retries on `SQLITE_BUSY`. A regression test counts file descriptors across 25 failed constructions, so the leak is caught without a sanitizer.
