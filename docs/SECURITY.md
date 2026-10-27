# Security review

A self-review of every source file against the threats below, then sanitizer runs, fuzz-style tests and mutation testing.

**Threats considered:** a local unprivileged user (squat on or read the IPC endpoint or database); a faulty or compromised worker (forge logs, bloat the database, impersonate); whoever can write the config (it decides which commands run); and malformed input anywhere. **Out of scope:** an attacker who is already the same user or root.

## Findings (all fixed unless noted)

| # | finding | fix |
|---|---|---|
| 1 | IPC socket at a predictable path in `/tmp` | private `mkdtemp` directory (0700), socket 0600 |
| 2 | no peer authentication: first connector was trusted | peer uid check, and the supervisor requires the spawned worker's pid |
| 3 | config could make the agent run anything, no trust check | owner and permission checks on the open descriptor, 1 MiB cap |
| 4 | commands resolved through `PATH` or the current directory | absolute paths only; `execve`, or an explicit application path on Windows |
| 5 | commands inherited the supervisor's environment | fixed minimal environment (POSIX only) |
| 6 | worker output trusted: forged log lines, huge fields, backdated times | id validation, sanitized and capped text, the supervisor's own clock |
| 7 | database created world-readable | created 0600 |
| 8 | a FIFO swapped in for a checked file blocked the worker | open non-blocking, require a regular file |
| 9 | the hang watchdog was off in one-shot mode | now always on |
| 10 | "database is locked" race on startup, still reproduced on Windows | busy timeout first, then retry on `SQLITE_BUSY`; deterministic test |
| 11 | SQLite handle leaked when the constructor threw (found by ASan) | owned by a `unique_ptr` with a custom deleter |
| 12 | stale socket directories after SIGKILL | cleanup that never follows symlinks |
| 13 | no compiler or linker hardening | added to the build (unverified build) |
| 14 | default Windows pipe permissions | explicit access-control list (unverified on Windows) |

## Open risks

- **No privilege separation:** the worker runs with the supervisor's privileges while parsing untrusted files. A real product would sandbox it.
- **Windows controls are unverified:** the pipe access-control list, impostor rejection and process-tree kill have no Windows tests yet. The peer-pid lookup works on its success path, since the handshake passes there.
- **Checks measure racily:** file checks follow symlinks and read paths a local attacker may be changing.
- **Config parent directory** permissions are not checked.
- **The other-user rejection test needs root** and skips otherwise.
- **Command output** (first 200 bytes of a failure) is logged and stored.
