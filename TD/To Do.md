- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
It _might_ be blocked, but not because it’s “hacky” — because some orgs lock down images and runtimes in a few common ways.

Here’s what could block it, and the safer alternatives.

## What could block that `/bin/sh -c "envsubst ... && nginx ..."` approach

1. **Your image might not include `envsubst`**
    

- `envsubst` usually comes from `gettext` / `gettext-base`.
    
- Many minimal images (including some nginx variants) don’t have it.
    
- In that case you’ll get: `envsubst: not found`.
    

2. **Security policies may restrict overriding the container command**
    

- Some platforms (or internal policies) require the image’s default `CMD/ENTRYPOINT` not be overridden.
    
- If you’re running plain Docker on WEnix, this is less common, but CI/K8s admission policies can enforce it.
    

3. **Read-only filesystem**
    

- If the container FS is read-only, you can’t write `/usr/share/nginx/html/env.js`.
    
- You’d see “read-only file system” errors.
    

4. **Running as non-root / permission issues**
    

- If nginx image runs as non-root and the html dir isn’t writable, writing `env.js` may fail.
    

None of these are guaranteed; they’re just the usual “org hardening” points.

---

## How to check quickly (no guessing)

Run this on WEnix:

```bash
docker run --rm -it thales-edge-ui /bin/sh -c "which envsubst || echo 'no envsubst'; id; ls -ld /usr/share/nginx/html"
```

Interpretation:

- If it prints `no envsubst` → you need the alternative below.
    
- If `/usr/share/nginx/html` isn’t writable → you need to write somewhere else (or bake it at build time).
    
- If `/bin/sh` doesn’t exist → image is super minimal; then this method won’t work.
    

---

## If your org blocks it: best alternatives

### ✅ Alternative A (most org-friendly): **Bake `env.js` at build time**

If your PSK is not meant to be secret (dev-only), you can build an image per environment.  
Downside: you must rebuild when PSK changes.

### ✅ Alternative B (most enterprise-safe): **Don’t send PSK to Angular at all**

Keep `DEEPPHAVEN_PSK` only in backend, and UI calls backend.  
This avoids leaking PSK to every browser.

### ✅ Alternative C (still runtime, but less “command override”): entrypoint script

Yes it’s “extra work”, but it’s also the most standard pattern and least likely to be blocked, because:

- you don’t override command at runtime
    
- it’s part of the image
    
- security scanners understand it
    

---

## Biggest non-org issue: PSK exposure

Even if your org allows it, injecting `DEEPPHAVEN_PSK` into browser JS means any user can see it.

If that PSK gives access to anything important, your security team _will_ care.

---

If you paste the output of this command:

```bash
docker inspect --format='Entrypoint={{.Config.Entrypoint}} Cmd={{.Config.Cmd}} User={{.Config.User}}' thales-edge-ui
```

…I can tell you which method is most likely to work in your org **without** getting blocked.