- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Got it — **this time it’s not a “can’t connect” problem**. The container is **actually dying**.

In your screenshots:

- Deephaven prints: `Server started on port 10000` and `Press Control-C to exit`
    
- Then immediately: **`Aborted!`**
    
- After that, `curl` fails (because there’s nothing listening)
    
- And `docker exec ...` says container is not running
    

So we need to find **why the Deephaven process is exiting with SIGABRT**.

---

## 1) Check the real exit reason (this will tell us what happened)

Run:

```bash
docker ps -a | grep deephaven
docker inspect deephaven --format='ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}} FinishedAt={{.State.FinishedAt}}'
```

**Interpretation**

- `OOMKilled=true` → it’s memory; we fix by increasing memory/shm
    
- `ExitCode=134` → classic SIGABRT (native/JVM abort). We need crash logs.
    

---

## 2) Get the last logs right before it aborts

```bash
docker logs deephaven --tail 200
```

(If the container is gone, rerun it, wait for it to abort, then run the above.)

---

## 3) Run it detached and capture JVM crash logs to a folder

This makes debugging deterministic:

```bash
mkdir -p /shared-resources/deephaven-crash

docker rm -f deephaven 2>/dev/null || true

docker run -d --name deephaven \
  -p 10000:10000 \
  --shm-size=1g \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK=my-fixed-psk \
  -e JAVA_TOOL_OPTIONS='-XX:ErrorFile=/tmp/hs_err_pid%p.log -XX:HeapDumpPath=/tmp' \
  -v /shared-resources/deephaven-crash:/tmp \
  deephaven-local:1.1.0
```

Then watch:

```bash
docker logs -f deephaven
```

If it aborts, check for crash files:

```bash
ls -lh /shared-resources/deephaven-crash | head
```

You’ll likely see `hs_err_pid*.log` (JVM fatal error report). That file will tell us the exact cause.

---

## 4) Quick sanity: confirm it ever listens before dying

Right after starting it (while it’s still “Up”):

```bash
docker exec -it deephaven bash -lc "ss -lntp | grep 10000 || true"
```

---

### Most common causes of this exact “Aborted!” pattern in containers

- **JVM native crash** (hs_err log will confirm)
    
- **Shared memory too small** (`--shm-size=1g` often fixes)
    
- **Resource limits / cgroup memory pressure** (inspect will show OOMKilled)
    

---

### Send me these 2 outputs and I’ll tell you the exact fix

```bash
docker inspect deephaven --format='ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}}'
```

```bash
docker logs deephaven --tail 200
```

If an `hs_err_pid*.log` file appears, paste the **top ~30 lines** of it (that section contains the smoking gun).