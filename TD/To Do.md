Short answer: **Yes — very likely this `node:20.19-bullseye` image may be blocked on a corporate machine** ❗  
But **you still have clean, compliant alternatives** that keep your setup intact.

I’ll break this down clearly and give you **safe options**.

---

## Why `node:20.19-bullseye` might be blocked

In most enterprise environments (WEnix / corporate laptops):

- ❌ **Docker Hub images** (`node:*`, `nginx:*`, `ubuntu:*`) are often **blocked**
    
- ❌ Pulling public images may be disabled
    
- ❌ Security scanners may reject Debian-based images
    
- ✅ **Internal Nexus / Artifactory images ARE allowed**
    
- ✅ **RedHat UBI images ARE allowed**
    

So yes — **this command may fail** on your org machine:

```bash
node:20.19-bullseye
```

---

## ✅ What DOES work in corporate environments

### OPTION 1 — Use your **already-approved Angular image** (BEST)

You already built:

```
thales-edge:latest
```

That image:

- Uses **UBI**
    
- Has **Node 20 installed via tarball**
    
- Is **org-compliant**
    

So instead of pulling `node:20.19-bullseye`, do this 👇

```bash
docker run --name thales-edge-dev --rm -it \
  --network dh-net \
  -p 4200:4200 \
  -v "%cd%":/app \
  -w /app \
  thales-edge:latest \
  bash -lc "npm ci && npx ng serve --host 0.0.0.0 --port 4200 --proxy-config proxy.conf.json"
```

✅ No public image  
✅ No Node upgrade on host  
✅ Fully compliant  
✅ Same dev behavior

---

### OPTION 2 — Use **internal Nexus Node image** (if available)

If your org has something like:

```
nexus.td.com/node:20
```

Then use:

```bash
FROM nexus.td.com/node:20
```

or at runtime:

```bash
nexus.td.com/node:20
```

But this depends on what **your Nexus actually hosts**.

---

### OPTION 3 — Use **Angular PROD mode only (no ng serve)**

If Dev mode is too restricted, fall back to:

```bash
npm run build
docker build -t thales-edge .
docker run -d -p 4200:8080 thales-edge
```

No Node needed at runtime.  
This is the **cleanest enterprise-safe approach**.

---

## ❌ What will NOT work

|Approach|Why|
|---|---|
|Installing Node 20 on host|❌ Restricted|
|Using `node:*` from Docker Hub|❌ Blocked|
|Running `ng serve` on host|❌ Node version mismatch|
|Browser → Deephaven directly|❌ CORS + DNS|

---

## Final recommendation (what YOU should do)

Since you already **successfully built `thales-edge` with Node 20**:

👉 **Use that image for dev-mode `ng serve`**

```bash
thales-edge:latest
```

This keeps:

- your **proxy config**
    
- your **Angular Dev UX**
    
- your **corporate compliance**
    

---

If you want, next I can:

- give you a **single unified script** (`run-dev.sh`)
    
- or help you **convert dev proxy → nginx proxy** so you never need `ng serve` again
    

Just tell me 👍