- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Got it — and the good news is: you can keep **exactly the same workflow** (PSK + allowed origins + port) in your **Docker-on-WEnix** setup. You’ll just express those “environment + run deephaven” steps as **container env vars + docker run args** (or a wrapper script).

### 1) The equivalent of “activate venv + export env + run deephaven”

In Docker, you don’t “activate”; you pass env vars to the container.

Run like this:

```bash
docker run --rm --name deephaven \
  -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK="your-psk-here" \
  -e DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200" \
  -e DEEPHAVEN_DATA_DIR=/data \
  -e DEEPHAVEN_CACHE_DIR=/cache \
  -e DEEPHAVEN_CONFIG_DIR=/config \
  -v deephaven-data:/data \
  -v deephaven-cache:/cache \
  -v deephaven-config:/config \
  deephaven-local:1.0
```

That covers what you were doing on Windows with:

- `DEEPHAVEN_AUTH_TYPE=psk`
    
- allowed origins for Angular
    
- data/cache/config dirs
    

> If your Angular is not running on the same host as the container, replace `localhost` with the actual host/IP you access from.

---

### 2) “Get the PSK”

In Docker you have two options:

**Option A (recommended): you choose the PSK**  
Just set it yourself (`DEEPHAVEN_PSK="..."`) and reuse it in:

- Angular `.env` / environment.ts
    
- backend `application.yml`
    

**Option B: generate a PSK and print it**  
If you want the same “print PSK and copy paste” flow, do it outside the container once:

```bash
openssl rand -hex 16
```

Then pass that value as `DEEPHAVEN_PSK`.

(Containers shouldn’t auto-generate secrets unless you store them somewhere.)

---

### 3) Backend + Frontend config stays the same idea

You were doing:

- Start Deephaven → get PSK
    
- Put PSK into UI env + backend yml
    

You’ll still do that, except now Deephaven PSK is **controlled by docker run** (or compose), so it’s stable and shareable.

Example backend (Spring) idea:

```yaml
deephaven:
  url: http://<wenix-host>:10000
  psk: ${DEEPHAVEN_PSK}
```

Example Angular idea:

- `DEEPhAVEN_URL=http://<wenix-host>:10000`
    
- `DEEPhAVEN_PSK=...`
    

---

### 4) One important gotcha: “allowed origins”

If your Angular is served from **another port** (say 4200) and Deephaven is on 10000, the origin must match exactly.

If you access the UI using a hostname (not localhost), include that too:

```bash
-e DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200,http://<wenix-host>:4200"
```

---

### 5) If you want the cleanest “Windows-like” flow: make a `start-deephaven.sh`

Create `start-deephaven.sh` in WEnix:

```bash
#!/usr/bin/env bash
set -euo pipefail

export DEEPHAVEN_PSK="${DEEPHAVEN_PSK:-$(openssl rand -hex 16)}"
echo "DEEPHAVEN_PSK=$DEEPHAVEN_PSK"

docker run --rm --name deephaven \
  -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK="$DEEPHAVEN_PSK" \
  -e DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200" \
  -v deephaven-data:/data \
  deephaven-local:1.0
```

Then you run:

```bash
chmod +x start-deephaven.sh
./start-deephaven.sh
```

It prints the PSK like you’re used to.

---

If you tell me what **exactly** your backend uses to talk to Deephaven (REST? websocket? deephaven-java-client? python client?), I’ll map the exact config keys/headers you should set so the PSK is used correctly end-to-end.