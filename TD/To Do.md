- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Short answer: **Yes, you _can_** — but **not as-is**.  
`registry.access.redhat.com/ubi8/ubi` is a **minimal RHEL base**, so you must install **Python + system deps yourself** before Deephaven will work.

Below is the **correct, enterprise-safe way** to do it.

---

## Why `ubi8/ubi` needs extra steps

- UBI **does NOT include Python**
    
- No `pip`, no `venv`
    
- Deephaven needs:
    
    - Python 3.9/3.10
        
    - `pip`
        
    - native libs (glibc, openssl, libstdc++, etc.)
        

So we bootstrap Python using **dnf**.

---

## ✅ Recommended Dockerfile (UBI 8 + offline wheels)

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

# Make python/pip available as "python" and "pip"
RUN alternatives --set python /usr/bin/python3.9 \
 && alternatives --set pip /usr/bin/pip3.9

# ---- App setup ----
WORKDIR /app
COPY wheels/ /wheels/

RUN pip install --no-index --find-links=/wheels deephaven-server

EXPOSE 10000

CMD ["python", "-m", "deephaven_server"]
```

---

## 🔑 Important notes (read this)

### 1) Wheels must be **Linux wheels**

When you download wheels on Windows, make sure they are **manylinux** wheels, not Windows ones.

Best practice:

```bash
pip download deephaven-server \
  --platform manylinux2014_x86_64 \
  --python-version 39 \
  --only-binary=:all: \
  -d wheels
```

Do this on **any machine that has internet** (Windows is fine).

---

### 2) If build fails on native deps

If you see errors like:

- `gcc not found`
    
- `cannot find -lstdc++`
    
- `libssl.so not found`
    

Just add the missing package via `dnf install`.

UBI is enterprise-friendly but intentionally minimal.

---

## 🟢 Why UBI is a GOOD choice in your org

- Approved RedHat registry
    
- Security-scanned
    
- Commonly allowed even when DockerHub/GHCR are blocked
    
- Matches enterprise Linux expectations (RHEL-like)
    

---

## When this approach is ideal vs not

|Situation|Verdict|
|---|---|
|Docker allowed only in WEnix|✅ Perfect|
|No external registry pulls|✅ Works|
|Need reproducible team setup|✅ Good|
|Want “official Deephaven image”|❌ Not possible|

---

## If you want next

I can:

- Slim this image (multi-stage)
    
- Add PSK auth
    
- Add your `orchestrator.py`
    
- Add startup scripts
    
- Turn this into a `docker-compose` service alongside your backend
    

Just say the word 👍