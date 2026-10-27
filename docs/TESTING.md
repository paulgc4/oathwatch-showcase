# Testing and verification

## Results

| check | result |
|---|---|
| unit tests | 85 (Linux) |
| integration tests (real binaries, plus a misbehaving fake worker) | 27 (Linux) |
| AddressSanitizer, UBSan, LeakSanitizer on the full suite | clean, after one real finding |
| line and branch coverage (gcov, Linux) | 92.4% and 57.1% (exception paths lower the branch figure) |
| mutation testing | 20 deliberate breakages: 19 caught by the suite, 1 only under ASan |
| Windows (MSVC, CMake, vcpkg) | tests not guarded as POSIX-only pass; the first run failed 1 of 63 and led to a fix |

## How these numbers were produced

- **Linux results** (counts, sanitizers, coverage, mutation testing) were produced during development in a Linux VM.
- **Windows results** are from a real Windows machine.
- There is **no public CI**: the source is private, so results are self-reported. The mutation-testing script is a local tool and is not published. Source available on request.

## What testing found

- A startup race ("database is locked") from the integration tests; its first fix still failed on Windows, where SQLite can return "busy" without waiting.
- A SQLite handle leaked when a constructor threw, found by AddressSanitizer.
- A hang watchdog that was off in one-shot mode, found by reading coverage gaps.
- A test that passed only by luck: deleting the zero-length-frame guard still passed because a second check rejected the same bytes after an out-of-bounds read. Only ASan caught it, which is why a sanitizer job belongs in CI.

## Not verified

- Windows pipe access control, impostor rejection, hostile-worker defences and process-tree kill (those tests run on POSIX only so far).
- macOS; syscall failure paths (interrupted calls, short writes, `fork` failure); and any independent audit.
