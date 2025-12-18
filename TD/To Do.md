- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Perfect — the error in your screenshot is **clear and fixable** 👍  
You’re **very close**.

---

## ❌ What’s failing (root cause)

This line is failing:

```dockerfile
RUN alternatives --set python /usr/bin/python3.9 \
 && alternatives --set pip /usr/bin/pip3.9
```

Error:

```
alternatives: cannot access /usr/bin/alternatives/pip: No such file or directory
```

### Why?

On **UBI**, `alternatives --set` only works **if an alternatives entry already exists**.

In UBI:

- `python` alternative ❌ does NOT exist by default
    
- `pip` alternative ❌ does NOT exist by default
    

So `--set` fails.

---

## ✅ Correct fixes (pick ONE)

---

## ✅ **Fix A (Recommended): Don’t use alternatives at all**

Just call **python3.9 / pip3.9 explicitly** — this is the cleanest and most reliable approach.

### 🔧 Fixed Dockerfile

```dockerfile
FROM registry.access.redhat.com/ubi8/ubi

# ---- System packages ----
RUN dnf install -y \
      python39 \
      python39-pip \
      gcc \
      gcc-c++ \
      make \
      libstdc++ \
      glibc-langpack-en \
      openssl \
    && dnf clean all

WORKDIR /app
COPY wheels/ /wheels/

# Use explicit pip
RUN pip3.9 install --no-index --find-links=/wheels deephaven-server

EXPOSE 10000

CMD ["python3.9", "-m", "deephaven_server"]
```

✅ **This will build cleanly**

---

## ✅ **Fix B: Properly register alternatives (if you really want `python`)**

Only do this if your org **requires** `python` instead of `python3.9`.

```dockerfile
RUN alternatives --install /usr/bin/python python /usr/bin/python3.9 1 \
 && alternatives --install /usr/bin/pip pip /usr/bin/pip3.9 1
```

Then `--set` works — but honestly, it’s unnecessary.

---

## 🚀 What to do now

From the folder with `Dockerfile` + `wheels/`:

```bash
docker build -t deephaven-local:1.0 .
```

If that succeeds:

```bash
docker run --rm -p 10000:10000 deephaven-local:1.0
```

Open:

```
http://<wenix-host>:10000/ide
```

---

## 🧠 Why this approach is enterprise-correct

- ✅ No DockerHub / GHCR pulls
    
- ✅ Uses RedHat UBI (approved)
    
- ✅ Fully offline wheels
    
- ✅ Reproducible for teammates
    
- ✅ No Windows artifacts involved
    

---

## If the next error appears…

Common next ones:

- Missing `libX11` / `fontconfig` → add via `dnf`
    
- Missing wheel dependency → rerun `pip download` with deps
    

Paste the **next error block** if it happens — you’re now in the final 10%.