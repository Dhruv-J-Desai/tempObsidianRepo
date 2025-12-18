- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Nice — you fixed the port mapping. ✅  
Now you’re getting **403 Forbidden**, which means:

- Nginx **is reachable**
    
- But it’s **not allowed to serve the directory / files** you copied (permissions / SELinux labeling / wrong doc root)
    

On **UBI nginx**, this is super common because it runs as a non-root user and expects content under specific paths/permissions.

---

## What’s causing 403 in your case

Most likely **your Angular files in `/usr/share/nginx/html` are not readable by the nginx user** (or nginx is serving a different root, like `/opt/app-root/src`).

Also your `docker ps` shows exposed ports `8080/tcp, 8443/tcp` which matches UBI nginx defaults.

---

## Step 1 — confirm what root nginx is serving

Run:

```bash
docker exec -it thales-edge-ui sh -c "grep -R \"root \" -n /etc/nginx/nginx.conf /etc/nginx/conf.d 2>/dev/null | head -50"
```

If it shows something like:

- `root /usr/share/nginx/html;` → then it’s **permissions**
    
- `root /opt/app-root/src;` (or similar) → then your COPY path is wrong for UBI nginx
    

---

## Step 2 — check if index.html exists where nginx expects

First check both places:

```bash
docker exec -it thales-edge-ui sh -c "ls -la /usr/share/nginx/html | head -30"
docker exec -it thales-edge-ui sh -c "ls -la /opt/app-root/src 2>/dev/null | head -30"
```

You should see `index.html` in the actual root folder.

---

## Fix option A (recommended): copy into UBI nginx expected web root + correct permissions

Many UBI nginx images expect content in:

✅ `/opt/app-root/src`

So update runtime stage to:

```dockerfile
FROM registry.access.redhat.com/ubi8/nginx-120

COPY --from=build /app/dist/thales-edge/browser/ /opt/app-root/src/

USER root
RUN chmod -R g+rwX /opt/app-root/src && chgrp -R 0 /opt/app-root/src
USER 1001
```

Then rebuild + run:

```bash
docker build -t thales-edge-ui .
docker rm -f thales-edge-ui
docker run -d -p 8080:8080 --name thales-edge-ui thales-edge-ui
```

---

## Fix option B: keep `/usr/share/nginx/html` but set permissions

If nginx root really is `/usr/share/nginx/html`, then do:

```dockerfile
COPY --from=build /app/dist/thales-edge/browser/ /usr/share/nginx/html/

USER root
RUN chmod -R a+rX /usr/share/nginx/html
USER 1001
```

(UBI images often run as `1001`, not `nginx`.)

---

## Why “403” specifically?

403 on nginx static hosting usually happens when:

- directory has no execute bit for traversal (`x`)
    
- files aren’t readable by the running user
    
- nginx has no `index` file or autoindex disabled (but then usually you see 403 too)
    

So permissions/root-path are the key.

---

### Do this one command and paste output (it will pinpoint instantly)

```bash
docker exec -it thales-edge-ui sh -c "nginx -T 2>/dev/null | sed -n '1,160p'"
```

From that I’ll tell you exactly which root path it’s using and which of the two fixes above is the right one.