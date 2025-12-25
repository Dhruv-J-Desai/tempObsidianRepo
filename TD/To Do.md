I see what’s happening from your screenshots:

- In your **compose**, you set:
    

```yaml
DEEPHAVEN_AUTH_TYPE: "psk"
DEEPHAVEN_PSK: "dev-psk"
```

- But in the **DeepHaven log** it says it’s running at:
    

`http://0.0.0.0:10000/?psk=shaouw7...`

So DeepHaven is **not reading `DEEPHAVEN_PSK`** from your compose. When it can’t find a PSK, it generates one and prints it in the URL.

This almost always comes down to **one of these 3 causes**:

---

## 1) Your image expects a different variable name (very likely in your custom `deephaven-local:*` image)

Some DeepHaven entrypoints expect **`DH_PSK`** (or similar) and then map it internally.

✅ Fix (make it bulletproof): set **both** env vars:

```yaml
deephaven:
  environment:
    DEEPHAVEN_AUTH_TYPE: psk
    DEEPHAVEN_PSK: dev-psk
    DH_PSK: dev-psk
```

Then restart cleanly (important):

```bash
docker compose down
docker compose up --build
```

---

## 2) Your container is not actually using the compose file you’re editing (very common)

Sometimes you’re tailing logs from a container started earlier with `docker run` or a different compose file.

✅ Verify what the running container _really_ has:

```bash
docker exec -it deephaven sh -lc 'printenv | egrep "DEEPHAVEN_AUTH_TYPE|DEEPHAVEN_PSK|DH_PSK"'
```

Expected output should include:

- `DEEPHAVEN_PSK=dev-psk` (and/or DH_PSK)
    

If it’s missing → your compose changes are not being applied to that container.

Also confirm the exact file being used:

```bash
docker compose config
```

This prints the fully-resolved compose (what Docker will actually run).

---

## 3) Your image entrypoint is overriding / generating PSK no matter what (custom script issue)

Your log shows:

- `/entrypoint.sh /etc...`
    
- plus the Deephaven URL printing a generated PSK
    

That suggests your custom image’s entrypoint script might be doing something like:

- `DEEPHAVEN_PSK=$(openssl rand ...)` even if one is provided
    
- or reading from a different variable and ignoring `DEEPHAVEN_PSK`
    

✅ Quick proof inside the container:

```bash
docker exec -it deephaven sh -lc 'echo "PSK=$DEEPHAVEN_PSK DH_PSK=$DH_PSK"; ps -ef | head -n 50'
```

If env shows `DEEPHAVEN_PSK=dev-psk` but the URL still shows a random one → the entrypoint is overriding it.

✅ Fix in that case: update the entrypoint logic to only generate when empty, like:

```sh
if [ -z "$DEEPHAVEN_PSK" ] && [ -z "$DH_PSK" ]; then
  export DEEPHAVEN_PSK="$(some_random)"
fi
```

---

# The fastest “make it work” solution

Put this in compose:

```yaml
deephaven:
  environment:
    DEEPHAVEN_AUTH_TYPE: psk
    DEEPHAVEN_PSK: dev-psk
    DH_PSK: dev-psk
```

Then do:

```bash
docker compose down
docker compose up
docker logs -f deephaven
```

You should now see the URL ending with:

`?psk=dev-psk`

---

## One more important note

Your `DEEPHAVEN_SERVER_ALLOWED_ORIGINS` includes `http://localhost:4200` — that’s fine for your Angular dev server. But once UI is containerized behind nginx, you may also need to allow `http://localhost:4200` (host-mapped) and/or `http://ui` etc. Not related to PSK, but useful.

---

If you paste your **deephaven Dockerfile** (or `entrypoint.sh`) from `deephaven-local:*`, I can point to the exact line that is generating the PSK and give you the exact change.