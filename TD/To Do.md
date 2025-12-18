- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Perfect — your screenshot gives the key facts:

- `docker inspect` shows **ExitCode=1** and **OOMKilled=false**
    
- Container **is not running** when you try `docker exec`
    
- So this is **not a network problem** and **not OOM** — the Deephaven process is **exiting**.
    

Also: that standalone `Aborted!` line is very often what you see when the _foreground process_ (or the terminal session) gets interrupted. But since the container is truly stopped, we’ll treat it as a real exit and capture _why_.

---

## 1) Get the exact exit info (includes FinishedAt + error)

Run:

```bash
docker inspect deephaven --format='ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}} StartedAt={{.State.StartedAt}} FinishedAt={{.State.FinishedAt}}'
```

---

## 2) Get _full_ logs (not `-f`, not tail)

The reason is usually **just above** the last lines you’re seeing.

```bash
docker logs deephaven
```

If it’s too long, do:

```bash
docker logs deephaven | tail -n 400
```

---

## 3) Run it in “crash-capture” mode so we can see JVM fatal logs if any

This is the fastest way to catch a JVM/native abort:

```bash
mkdir -p /shared-resources/deephaven-crash
docker rm -f deephaven 2>/dev/null || true

docker run -d --name deephaven \
  -p 10000:10000 \
  --shm-size=1g \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK=my-fixed-psk \
  -e JAVA_TOOL_OPTIONS='-XX:ErrorFile=/tmp/hs_err_pid%p.log -XX:HeapDumpPath=/tmp -XX:+UnlockDiagnosticVMOptions -XX:+LogVMOutput -XX:LogFile=/tmp/jvm.log' \
  -v /shared-resources/deephaven-crash:/tmp \
  deephaven-local:1.1.0
```

Then watch:

```bash
docker logs -f deephaven
```

If it exits again, check the crash folder:

```bash
ls -lh /shared-resources/deephaven-crash | head
```

- If you see `hs_err_pid*.log` → it’s a JVM fatal crash (we fix based on the first lines).
    
- If you see `jvm.log` → it’ll show the last JVM events before exit.
    

---

## 4) One very common “gotcha” in your screenshots

You’re running `docker logs -f deephaven` and the log says **“Press Control-C to exit”**.

That message is _from Deephaven_, not Docker — but if you press Ctrl+C while you’re in the same terminal session (or the session sends a hangup), it can stop the foreground attach/streaming and you end up thinking Deephaven “aborted”.

So: always run Deephaven detached (`-d`) (you already did) and only tail logs in a separate command.

---

### Send me these two things (and I’ll tell you the exact fix immediately)

1. Output of:
    

```bash
docker logs deephaven | tail -n 200
```

2. Output of:
    

```bash
ls -lh /shared-resources/deephaven-crash | head
```

If `hs_err_pid*.log` exists, paste the **first ~30 lines** of that file. That will pinpoint the cause in one shot.