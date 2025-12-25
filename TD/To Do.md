If `cacerts-updated` is **a file on your host**, but Java says **“Is a directory”**, then one of these is happening:

## What’s actually happening (most common)

### 1) Docker created a directory **inside the container** because your host path didn’t resolve

When you bind-mount a path and the **source path doesn’t exist** (or Docker can’t see it), Docker often ends up creating a **directory** at the target location.

So inside the container, your path becomes a folder → Java throws **Is a directory**.

This is super common in WEnix when:

- the path is wrong by 1 character
    
- you’re running compose from a different machine/context
    
- permissions prevent Docker from reading the file
    
- you used a relative path that resolves differently
    

---

## Prove it in 10 seconds

Run these **inside WEnix** (host), and then **inside the container**.

### On WEnix host

```bash
ls -l /shared-resources/stack/bishowcase-backend/cacerts-updated
file /shared-resources/stack/bishowcase-backend/cacerts-updated
```

You should see something like: `regular file` (or `Java KeyStore`).

### Inside the container

```bash
docker exec -it bishowcase-backend sh -lc "ls -l /opt/app/shared-resources/stack/bishowcase-backend/cacerts-updated && file /opt/app/shared-resources/stack/bishowcase-backend/cacerts-updated"
```

If it prints `directory`, then the mount didn’t mount the file correctly.

---

## The fix (do this even if filename has no .jks)

Mount file-to-file explicitly (best practice):

```yaml
bishowcase-backend:
  volumes:
    - /shared-resources/stack/bishowcase-backend/cacerts-updated:/run/secrets/truststore.jks:ro
  environment:
    SSL_TRUSTSTORE_LOCATION: /run/secrets/truststore.jks
    SSL_TRUSTSTORE_PASSWORD: "${SSL_TRUSTSTORE_PASSWORD}"
```

Why this works: it forces Docker to treat the target as a **file**, not a directory path.

---

## Two extra gotchas to check

### 2) You accidentally added a trailing slash somewhere

If you mount like:

- `.../cacerts-updated/:/opt/app/.../cacerts-updated`  
    Docker treats it like a **directory**.
    

So make sure **no trailing slash** on the source.

### 3) `cacerts-updated` is a file, but you mounted the parent folder and then pointed env to the folder

If your compose is mounting the folder `.../bishowcase-backend/` and inside it there is `cacerts-updated`, then the env must point to the file:

✅ `/opt/app/.../cacerts-updated` (file)  
❌ `/opt/app/.../cacerts-updated/` (directory)

---

If you paste your **exact compose snippet** for `bishowcase-backend` (the `volumes:` + `environment:` lines), I’ll point out the exact reason it became a directory and give the corrected version.