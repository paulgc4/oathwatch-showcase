# Excerpt: timeouts that kill the whole tree

The core loop of `run_command`, which runs a check command with a deadline. From `src/process_posix.cpp` (excerpt only).

```cpp
  for (;;) {
    if (!eof) {
      pollfd p{fds[0], POLLIN, 0};
      int wait_ms = static_cast<int>(std::max<int64_t>(0, std::min<int64_t>(ms_until(deadline), 50)));
      ::poll(&p, 1, wait_ms);
      drain();
    } else {
      std::this_thread::sleep_for(ms(10));
    }
    pid_t w = ::waitpid(pid, &status, WNOHANG);
    if (w == pid) {
      drain();
      break;
    }
    if (Clock::now() >= deadline) {
      ::kill(-pid, SIGKILL);
      ::kill(pid, SIGKILL);
      while (::waitpid(pid, &status, 0) < 0 && errno == EINTR) {}
      r.timed_out = true;
      drain();
      break;
    }
  }
```

**Notice**

- It does **not** read until EOF. A background process started by the command inherits the pipe, so EOF may never arrive. The loop polls, checks `waitpid(WNOHANG)`, and enforces a deadline.
- `kill(-pid, SIGKILL)` kills the **process group**. Each spawned process leads its own group, so a timeout takes down everything the command started. Windows does the same with a job object set to kill on close.
- Two tests pin it down: one confirms the call returns promptly when a grandchild holds the pipe, and one records the grandchild's pid and asserts it is dead afterwards. A mutation that killed only the command's own process was caught.
