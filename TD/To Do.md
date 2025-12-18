- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Nice — from here it’s just **build → run → verify**, with one small gotcha: your `pip download ...` command must include **all dependencies**, not just the top package name.

### 1) Make sure your `wheels/` folder has deps too

In the folder that contains your Dockerfile, re-run the download with deps included (and pin pip to avoid surprises):

```bash
rm -rf wheels
mkdir wheels

pip download deephaven-server \
  --dest wheels \
  --platform manylinux2014_x86_64 \
  --python-version 39 \
  --only-binary=:all: \
  --implementation cp \
  --abi cp39 \
  --prefer-binary
```

If this pulls a lot of wheels, that’s good — it means deps are included.

---

### 2) Build the Docker image (in WEnix)

From the same folder as the Dockerfile:

```bash
docker build -t deephaven-local:1.0 .
```

If the build fails, copy the **first error block** (usually it’ll say which dependency is missing).

---

### 3) Run it

```bash
docker run --rm --name deephaven \
  -p 10000:10000 \
  deephaven-local:1.0
```

---

### 4) Verify it’s up

On WEnix:

```bash
docker logs -f deephaven
```

From a browser (depending on where you’re running):

- If you have browser access on that machine: `http://localhost:10000/ide`
    
- If you need to access from Windows: `http://<wenix-hostname-or-ip>:10000/ide`
    

If Windows can’t reach it, it’s usually firewall / routing.

---

## Two common issues and fixes

### A) `pip install` inside Docker says “No matching distribution found”

That means your `wheels/` folder is incomplete (missing deps) or contains wrong platform wheels.  
Re-run Step 1 exactly.

### B) UBI can’t `dnf install python39`

Some corporate networks block RedHat repos. If that happens, you have 2 options:

- use your org’s **internal yum repo** (if configured), or
    
- switch base image to one you _can_ `dnf` from (sometimes `ubi8/ubi-minimal` is even harder)
    

If you hit this, paste the `dnf` error lines.

---

If you share the last ~30 lines of `docker build` output (especially the red error section), I’ll tell you exactly what to tweak (packages, Python version, missing wheels, etc.).