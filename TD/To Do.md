- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Yep — and this is the _exact_ Docker networking trap:

### Why Windows browser can use `localhost`, but your **container** can’t

You have **three different “localhost” worlds**:

1. **Windows browser**
    

- `http://localhost:10000` means **your Windows machine**
    
- It works because the Deephaven port is exposed/forwarded to Windows (Docker Desktop, SSH tunnel, or port mapping)
    

2. **WEnix host (Linux)**
    

- `localhost:10000` means **the WEnix host**
    
- It works because Deephaven is published on the host (e.g., `-p 10000:10000`)
    

3. **dh-orchestrator container**
    

- `localhost:10000` means **inside the dh-orchestrator container itself**
    
- ❌ Deephaven is _not_ running inside that container → **connection refused**
    

So the fact that Windows browser can reach Deephaven with `localhost` does **not** mean your container can.

---

## Prove it (run the check _inside_ the orchestrator container)

Right now your “CONNECTED” check looks like it was run on the host. Do it inside the container:

```bash
docker exec -it dh-orchestrator sh -c 'echo > /dev/tcp/localhost/10000 && echo CONNECTED || echo FAILED'
```

Expected:

- likely `FAILED` (because localhost in that container ≠ host)
    

Now try using the **host gateway**:

```bash
docker exec -it dh-orchestrator sh -c 'echo > /dev/tcp/host.docker.internal/10000 && echo CONNECTED || echo FAILED'
```

If that gives `CONNECTED`, you’ve confirmed the fix.

---

## Fix option 1: Deephaven is running on the WEnix host (or published to host)

Run orchestrator with host-gateway mapping and point to it:

```bash
docker run -d --name dh-orchestrator \
  -p 8081:8081 \
  --add-host=host.docker.internal:host-gateway \
  -e DEEPHAVEN_HOST=host.docker.internal \
  -e DEEPHAVEN_PORT=10000 \
  -e DH_PSK=<your_psk> \
  dh-orchestrator:latest
```

✅ This makes the container reach “the host’s localhost”.

---

## Fix option 2 (better): Deephaven is another container

Put both containers on the same Docker network and use the **container name**:

```bash
docker network create dh-net

docker run -d --name deephaven --network dh-net \
  -p 10000:10000 \
  deephaven-local:1.2.0

docker run -d --name dh-orchestrator --network dh-net \
  -p 8081:8081 \
  -e DEEPHAVEN_HOST=deephaven \
  -e DEEPHAVEN_PORT=10000 \
  -e DH_PSK=<your_psk> \
  dh-orchestrator:latest
```

Then inside orchestrator, you use `deephaven:10000` (not localhost).

---

## What to change in your app config

Wherever you currently have:

- `DEEPHAVEN_HOST=localhost`  
    change it to:
    
- `host.docker.internal` (if Deephaven is via host port)  
    **or**
    
- `deephaven` (if Deephaven is another container on same network)
    

---

If you tell me **how Deephaven is running** right now:

- “as a docker container named ___” **or**
    
- “directly on host”  
    …I’ll give you the exact final `docker run` line for your setup (no guessing).