- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
That pattern is **super consistent** with one thing:

- With `-d` (detached), the container keeps running **only if the main process stays in the foreground**.
    
- If your Deephaven process (or your startup “app” script) **exits immediately**, Docker considers the container “done” and it stops.
    
- With `--rm -it`, it _looks_ “up” because you’re attached and seeing output, but the real question is: does the **Deephaven process stay alive**, or are you just sitting in an interactive shell / different entrypoint?
    

Let’s make this deterministic.

---

## 1) Run detached **and immediately check exit code + logs**

Do:

```bash
docker rm -f deephaven 2>/dev/null || true

docker run -d --name deephaven \
  -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK=my-fixed-psk \
  -e START_OPTS="-Ddeephaven.application.dir=/app.d" \
  -v /shared-resources/deephaven/app.d:/app.d:ro \
  deephaven-local:1.3.0

sleep 2
docker ps -a --filter name=deephaven
docker inspect deephaven --format='ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}}'
docker logs deephaven --tail 300
```

**This will tell us exactly why it stops.**

Most likely you’ll see your `orchestrator.py` throwing an exception, and Deephaven exits.

---

## 2) Important: your `.app` startup script can crash the whole server

If the python app script errors during startup, Deephaven may fail fast (container exits in `-d`).

To confirm, temporarily run Deephaven **without** application mode:

```bash
docker rm -f deephaven 2>/dev/null || true

docker run -d --name deephaven \
  -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK=my-fixed-psk \
  deephaven-local:1.3.0

sleep 2
docker ps --filter name=deephaven
docker logs deephaven --tail 80
```

- If this stays up ✅ → your **app mode script is the culprit**
    
- If it still exits ❌ → the problem is Deephaven startup itself (less likely since you had it running earlier)
    

---

## 3) The “--rm -it works” case: make sure you’re not accidentally running a shell

Many times people do:

```bash
docker run --rm -it image bash
```

…and that “stays up” because `bash` is running, not Deephaven.

So: what exact `--rm -it` command are you running?  
(You don’t need to answer with a question—just paste it here.)

---

## 4) Best practice: run orchestrator **after** server is up (so it can’t kill startup)

If your orchestrator is something that can fail (network calls, databricks creds, etc.), don’t run it as “startup app”. Instead:

### Start Deephaven normally (detached)

```bash
docker rm -f deephaven 2>/dev/null || true
docker run -d --name deephaven \
  -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK=my-fixed-psk \
  deephaven-local:1.3.0
```

### Then run orchestrator **manually** (or as a second container)

```bash
docker exec -it deephaven bash -lc "python3.9 /app.d/orchestrator.py"
```

This way: if `orchestrator.py` crashes, it won’t bring down the server.

---

## What I need from you (one paste)

Paste the output of this (it’s the fastest way to finish):

```bash
docker logs deephaven --tail 300
```

If the container exited, also paste:

```bash
docker inspect deephaven --format='ExitCode={{.State.ExitCode}} Error={{.State.Error}}'
```

Once I see that log, I’ll tell you whether:

- your `.app` file format is wrong,
    
- the script is failing on import,
    
- the script is exiting immediately,
    
- or Deephaven is configured to shut down after running the app.