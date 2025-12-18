- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Nice — the important part is: **it actually started successfully**. In your screenshot I can see:

- “Starting Deephaven server…”
    
- “Server started on port 10000”
    
- “Deephaven is running at [http://0.0.0.0:10000/?psk=…”](http://0.0.0.0:10000/?psk=%E2%80%A6%E2%80%9D)
    
- Then it shows **“Aborted”**
    

That “Aborted” is almost never Deephaven crashing by itself — it usually means **your terminal sent an interrupt / the parent shell got killed**, or Docker got stopped (Ctrl+C / session ended / WSL terminal closed / etc.). Deephaven was running fine up to that point.

### 1) Run it in detached mode so it doesn’t die when you close the terminal

```bash
docker run -d --name deephaven \
  -p 10000:10000 \
  deephaven-local:1.1.0
```

Check it’s running:

```bash
docker ps | grep deephaven
docker logs -f deephaven
```

Stop it when you want:

```bash
docker stop deephaven
```

---

### 2) If you _need_ the PSK, don’t rely on the auto-generated `?psk=...`

Auto PSK is annoying because it changes each run. Make it stable:

```bash
docker run -d --name deephaven \
  -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK="my-fixed-psk-123" \
  deephaven-local:1.1.0
```

Then you always use the same PSK in:

- Angular env
    
- backend `application.yml`
    

---

### 3) Confirm it’s reachable

From the WEnix host:

```bash
curl -I http://localhost:10000/ | head
```

From your Windows browser (if needed):

- `http://<wenix-hostname-or-ip>:10000/ide`
    

If Windows can’t reach it, it’s firewall/routing, not Deephaven.

---

### 4) Why you saw “Aborted”

Most common causes:

- You pressed **Ctrl+C**
    
- Terminal session disconnected
    
- WSL shell closed / crashed
    
- Docker daemon restarted
    

Running detached (`-d`) avoids all of these.

---

If you run it detached and it still stops, send me:

```bash
docker ps -a | grep deephaven
docker logs deephaven --tail 200
```

and I’ll pinpoint the real abort reason.