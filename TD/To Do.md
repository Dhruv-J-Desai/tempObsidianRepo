Perfect — that actually confirms the diagnosis.

Being on **`bridge`** means:

- ✅ they _can_ talk, **but only reliably by IP**, and name/DNS between containers is not guaranteed like it is on a **user-defined network**
    
- ❌ using `localhost` will **never** work for container→container
    

So the clean fix is: **create your own network and attach both containers** (no rebuild needed).

---

## Option A (fastest, no recreate): connect existing containers to a new network

### 1) Create network

```bash
docker network create dh-net
```

### 2) Attach both running containers to it

```bash
docker network connect dh-net deephaven-local
docker network connect dh-net dh-orchestrator
```

### 3) Verify they’re both attached

```bash
docker network inspect dh-net | grep -E "deephaven-local|dh-orchestrator"
```

---

## Option B (cleanest): recreate them directly on `dh-net`

(if you don’t mind restarting)

```bash
docker rm -f dh-orchestrator deephaven-local || true
docker network create dh-net || true

docker run -d --name deephaven-local --network dh-net \
  -p 10000:10000 -p 10001:10001 \
  deephaven-local:1.2.0

docker run -d --name dh-orchestrator --network dh-net \
  -p 8081:8081 \
  -e DH_HOST=deephaven-local \
  -e DH_PORT=10000 \
  -e DH_PSK=<psk> \
  dh-orchestrator:1.0
```

---

# Now the MOST IMPORTANT step: update orchestrator to use container name, not localhost

Right now your orchestrator is using `localhost:10000`.

Change to:

- `DH_HOST=deephaven-local`
    
- `DH_PORT=10000`
    

If you’re currently passing only `DH_PORT`, you **must** also pass host (or set it in `application.yml`).

### If you can restart the orchestrator container:

```bash
docker rm -f dh-orchestrator

docker run -d --name dh-orchestrator --network dh-net \
  -p 8081:8081 \
  -e DH_HOST=deephaven-local \
  -e DH_PORT=10000 \
  -e DH_PSK=<psk> \
  dh-orchestrator:1.0
```

---

## Quick connectivity test (inside orchestrator)

Now this should CONNECT:

```bash
docker exec -it dh-orchestrator sh -c "curl -v http://deephaven-local:10000/ 2>&1 | head -30"
```

Even if the HTTP response is 404/redirect/etc, that’s fine — we only care that it connects.

---

## Why this works

- On a **user-defined bridge network** (`dh-net`), Docker provides **built-in DNS**:
    
    - `deephaven-local` resolves automatically to the Deephaven container IP
        
- On default `bridge`, that DNS behavior is not dependable
    

---

If after you restart orchestrator it still calls `localhost`, paste the exact line in your logs where it prints Deephaven host/port, and I’ll tell you which property name your app is actually reading (so you set the correct `-e ...`).