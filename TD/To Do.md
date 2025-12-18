- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Yep — this screenshot shows you _fixed the caching_, but now we learned something new:

> The file **`/usr/local/bin/deephaven-server` does not exist** in your image.

So the issue is **not Docker anymore** — it’s that the CLI executable name/path you’re trying to run isn’t what got installed.

Also, based on your Windows workflow (`deephaven server --port 10000`), the CLI you actually want is usually **`deephaven`**, not `deephaven-server`.

Let’s confirm what got installed inside the image, then set the right `CMD`.

---

## 1) Inspect inside the image (override entrypoint so it can start)

Run this (it bypasses your failing CMD):

```bash
docker run --rm -it --entrypoint bash deephaven-local:1.0 -lc '
set -e
echo "== pip packages ==";
python3.9 -m pip list | grep -i deephaven || true;

echo "== which commands ==";
command -v deephaven || true;
command -v deephaven-server || true;

echo "== list /usr/local/bin ==";
ls -l /usr/local/bin | grep -i deephaven || true;

echo "== list /usr/bin ==";
ls -l /usr/bin | grep -i deephaven || true;
'
```

### What you’re looking for:

- If you see `/usr/local/bin/deephaven` (or `/usr/bin/deephaven`) → that’s the command to run.
    
- If you see `deephaven-server` somewhere else → we’ll use that exact path.
    

---

## 2) Most likely fix: run `deephaven server`

If the command exists as `deephaven`, update your Dockerfile to:

```dockerfile
CMD ["deephaven", "server", "--port", "10000"]
```

Then rebuild + run:

```bash
docker build --no-cache -t deephaven-local:2.0 .
docker run --rm -p 10000:10000 deephaven-local:2.0
```

---

## 3) If `deephaven` is not found either

Then your offline wheels likely installed only part of what you used on Windows.

In that case, rebuild your wheels including the UI/server tooling you used before (you mentioned these in your notes):

- `deephaven-server[kafka]`
    
- `deephaven-plugin-ui`
    

So regenerate wheels (on the machine where you can download):

```bash
rm -rf wheels && mkdir wheels
pip download "deephaven-server[kafka]" "deephaven-plugin-ui" \
  --dest wheels \
  --platform manylinux2014_x86_64 \
  --python-version 39 \
  --only-binary=:all: \
  --implementation cp \
  --abi cp39
```

Copy to WEnix, rebuild, then re-check with step (1).

---

Send me the output of the command in step **(1)** (just the few lines showing `command -v` and the `ls | grep` results) and I’ll tell you the exact final `CMD` line to use.