- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Got it — your error is:

```
sh: cannot create /dev/tcp/host.docker.internal/10000: Directory nonexistent
```

That tells us something **very specific**:

### ✅ Your container’s `sh` does NOT support `/dev/tcp/...`

`/dev/tcp` is a **bash feature** (and some shells), but not all `/bin/sh` implementations support it (and many enterprise images use a minimal `sh` that doesn’t).

So we need a different way to test connectivity **without nc** and **without /dev/tcp**.

---

# ✅ Best connectivity test you DO have: `curl`

Almost always present in UBI images. Try these **inside dh-orchestrator**:

### 1) Test Deephaven by container name (if Deephaven is another container)

```bash
docker exec -it dh-orchestrator sh -c "curl -sS -v http://deephaven:10000/ 2>&1 | head -40"
```

### 2) If Deephaven is on host: use host-gateway mapping, then curl it

First ensure you started orchestrator with this:

```bash
--add-host=host.docker.internal:host-gateway
```

Then test:

```bash
docker exec -it dh-orchestrator sh -c "curl -sS -v http://host.docker.internal:10000/ 2>&1 | head -40"
```

**Even if you get HTTP 404 or some message** — that’s still good.  
What you don’t want is: `Connection refused` / `Could not resolve host`.

---

# ✅ Another test that works even if curl is missing: Java itself

Since dh-orchestrator is a Java container, we can test TCP using Java:

```bash
docker exec -it dh-orchestrator sh -c 'java - << "EOF"
import java.net.*;
public class T {
  public static void main(String[] args) throws Exception {
    String host = args.length>0?args[0]:"deephaven";
    int port = args.length>1?Integer.parseInt(args[1]):10000;
    try (Socket s = new Socket()) {
      s.connect(new InetSocketAddress(host, port), 2000);
      System.out.println("CONNECTED to " + host + ":" + port);
    }
  }
}
EOF'
```

(If this is too heavy, we can do a simpler one-liner, but this works in locked-down images.)

---

# 🔥 Now the real fix (not just testing)

From your `docker ps`, I see:

- `deephaven-local:1.2.0` is running
    
- `dh-orchestrator:1.0` is running
    

But I don’t see a shared network.

## ✅ Fix: Put them on the same Docker network and use the container name

### 1) Create network

```bash
docker network create dh-net
```

### 2) Re-run Deephaven on that network

```bash
docker rm -f deephaven-local || true

docker run -d \
  --name deephaven-local \
  --network dh-net \
  -p 10000:10000 \
  deephaven-local:1.2.0
```

### 3) Re-run orchestrator on that network and point it to Deephaven

```bash
docker rm -f dh-orchestrator || true

docker run -d \
  --name dh-orchestrator \
  --network dh-net \
  -p 8081:8081 \
  -e DEEPHAVEN_HOST=deephaven-local \
  -e DEEPHAVEN_PORT=10000 \
  -e DH_PSK=<psk_here> \
  dh-orchestrator:1.0
```

✅ Now `DEEPHAVEN_HOST=deephaven-local` works reliably.

---

# Why your browser can use localhost but containers can’t

Browser `localhost` = your machine.  
Container `localhost` = itself.

So for container-to-container:  
✅ use container DNS name on a shared network.

---

## Quick question-free next step (do this now)

Run this to see if `curl` exists inside dh-orchestrator:

```bash
docker exec -it dh-orchestrator sh -c "curl --version || echo 'curl missing'"
```

If curl exists, we’ll use it to confirm after networking change. If not, we’ll use the Java socket test.

If you paste your current `docker run` commands for deephaven + dh-orchestrator, I’ll rewrite them into the final working set with the right ports and network.