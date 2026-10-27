# Design decisions

Each entry: the situation, the choice, and what it costs.

1. **Supervisor and worker are separate processes.** Evaluating checks runs arbitrary commands and parses arbitrary files, so a crash or hang there must not touch the state. *Cost:* an IPC protocol and restart logic. *Rejected:* one process with threads, where a parsing bug sits next to the data it can corrupt.

2. **A small hand-written length-prefixed protocol.** About 100 lines, no dependencies, readable in a hex dump. *Cost:* a hand-written parser is where bugs live, so it has property-style tests and runs under sanitizers. *Rejected:* JSON (ambiguity, weight) and protobuf for now.

3. **One interface, two OS transports.** `Channel` and `Listener` hide a Windows named pipe and a POSIX socket, so the supervisor and tests are platform-neutral. *Cost:* the Windows side is larger because timeouts need asynchronous I/O.

4. **Every wait has a deadline.** `receive(timeout)` and `accept(timeout)` everywhere, so the control loop can check for shutdown, a dead worker and silence twice a second.

5. **Kill process trees; never read until EOF.** A command can start background processes that hold its output pipe open forever. Each spawned process gets its own process group (a job object on Windows), and the loop polls against a deadline.

6. **The config file is code.** It must be a regular file owned by the user or root, not world-writable, checked on the open descriptor; commands need absolute paths and run with a fixed environment. *Cost:* stricter than a developer might expect, so group-writable files only warn.

7. **The worker is untrusted.** Ids are validated, text is sanitized and capped, the supervisor uses its own clock, and only the exact spawned process may connect. A fake worker in the test suite exercises this instead of assuming it.

8. **SQLite with WAL, a busy timeout plus retry, and an RAII handle.** Found and fixed a startup race and a handle leak; see the [excerpt](../excerpts/02_raii_and_constructor_exceptions.md).

9. **Restart with backoff, drain before failing, watchdog on silence.** Results sent just before a crash are kept, and a crash loop cannot spin the CPU.

10. **No gRPC for local IPC.** Protobuf for the schema is a clear next step. gRPC adds a heavy dependency, and on Windows local IPC it generally means loopback TCP, which weakens the access-control story. I would use gRPC on a future supervisor-to-server hop instead. It is not implemented.
