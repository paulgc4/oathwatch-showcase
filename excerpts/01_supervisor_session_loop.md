# Excerpt: the supervisor's session loop

The code that runs while one worker is alive: handshake, receive loop, and why a session ends. From `src/supervisor_main.cpp` (excerpt only). In the code the worker process is held as `child`.

```cpp
Outcome run_session(ow::Channel& chan, ow::ChildProcess& child, ow::StateStore& store,
                    const Options& opt, Stats& stats) {
  using namespace ow;
  std::optional<Message> m;
  try { m = chan.receive(3000); }
  catch (const ProtocolError& e) { log("ERROR", std::string("protocol error in handshake: ") + e.what()); return Outcome::Failed; }
  if (!m || m->type != MsgType::Hello || field(*m, "version") != std::to_string(kProtocolVersion)) {
    log("WARN", "no valid Hello from worker");
    return Outcome::Failed;
  }
  log("INFO", "worker connected, pid " + sanitize_text(field(*m, "pid"), 16));

  auto last_seen = Clock::now();
  const auto silence_limit = std::chrono::seconds(opt.silence_limit > 0 ? opt.silence_limit : 2 * opt.interval + 15);

  // Returns an outcome if this message ends the session.
  auto handle = [&](const Message& msg) -> std::optional<Outcome> {
    switch (msg.type) {
      case MsgType::CheckResult: handle_result(msg, store, stats); return std::nullopt;
      case MsgType::Heartbeat: return std::nullopt;
      case MsgType::Goodbye: log("INFO", "worker finished cleanly"); return Outcome::Finished;
      default: log("ERROR", "unexpected message type from worker"); return Outcome::Failed;
    }
  };

  while (!g_stop) {
    try { m = chan.receive(500); }
    catch (const ProtocolError& e) { log("ERROR", std::string("protocol error: ") + e.what()); return Outcome::Failed; }
    if (m) {
      last_seen = Clock::now();
      if (auto end = handle(*m)) return *end;
    } else if (chan.closed()) {
      log("WARN", "worker closed the connection");
      return Outcome::Failed;
    }
    if (!child.running()) {
      // Pick up anything the worker managed to send before it exited.
      for (;;) {
        std::optional<Message> d;
        try { d = chan.receive(0); } catch (const ProtocolError&) { break; }
        if (!d) break;
        if (auto end = handle(*d)) return *end;
      }
      log("WARN", "worker exited unexpectedly");
      return Outcome::Failed;
    }
    if (Clock::now() - last_seen > silence_limit) {  // also in --once mode: a hung worker must not block forever
      log("ERROR", "worker unresponsive, killing it");
      return Outcome::Failed;
    }
  }
  chan.send(Message{MsgType::Shutdown, {}});
  if (auto code = child.wait(2000)) log("INFO", "worker exited with code " + std::to_string(*code));
  else log("WARN", "worker did not exit after Shutdown; killing it");
  return Outcome::Stopped;
}
```

**Notice**

- Three outcomes: `Finished` (the worker said Goodbye), `Failed` (anything wrong), `Stopped` (the supervisor was asked to stop). `Failed`, and `Finished` in continuous mode, lead to a restart with backoff.
- Every failure is a distinct logged case, each with an integration test that uses a misbehaving fake worker.
- When the worker process has exited, the loop still reads messages it queued, so results sent just before a crash are kept.
- Silence is a failure: a worker is killed after `2 * interval + 15` seconds, in one-shot mode too. A version without that let a hung worker block a one-shot run forever; reading the coverage gaps found it.
- `receive(500)` returns twice a second, which is how shutdown, a dead worker and silence get checked.
