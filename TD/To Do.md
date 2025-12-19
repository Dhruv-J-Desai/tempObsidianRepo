- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Nice — you’re 90% there. The **only problem** is: in your current image (`ubi8/nginx-120`), the web root is **not** `/usr/share/nginx/html`.

From your Dockerfile, you copy Angular output to:

```
/opt/app-root/src/
```

So your `entrypoint.sh` must write `env.js` into **that** folder (and the template must also live there).

---

## 1) Fix `entrypoint.sh` paths for `ubi8/nginx-120`

Use this version:

```sh
#!/bin/sh
set -e

# RHEL nginx-120 serves content from /opt/app-root/src
DOCROOT="/opt/app-root/src"
TEMPLATE="$DOCROOT/assets/env.template.js"
TARGET="$DOCROOT/assets/env.js"

if [ ! -f "$TEMPLATE" ]; then
  echo "ERROR: Missing template: $TEMPLATE"
  echo "       Ensure env.template.js is copied into dist assets."
  exit 1
fi

if [ -z "$DEEPHAVEN_PSK" ]; then
  echo "WARNING: DEEPHAVEN_PSK is empty; env.js will contain an empty value."
  DEEPHAVEN_PSK=""
fi

# Replace placeholder with runtime value
sed "s|__DEEPHAVEN_PSK__|${DEEPHAVEN_PSK}|g" "$TEMPLATE" > "$TARGET"

# Start nginx in foreground (works in this image too)
exec nginx -g 'daemon off;'
```

✅ This matches your runtime image layout.

---

## 2) Update your Dockerfile (runtime stage)

In **Stage 2**, after copying the dist, also copy `entrypoint.sh` and ensure permissions.

Here’s the **full Stage 2** you should use:

```dockerfile
############################################
# Stage 2: Runtime Image (Nginx)
############################################
FROM registry.access.redhat.com/ubi8/nginx-120

# Copy Angular build output into nginx docroot
COPY --from=build /app/dist/thales-edge/browser /opt/app-root/src/

# Copy entrypoint into image
COPY entrypoint.sh /opt/app-root/entrypoint.sh

USER root
RUN chmod +x /opt/app-root/entrypoint.sh \
    && chgrp -R 0 /opt/app-root/src \
    && chmod -R g+rwX /opt/app-root/src \
    && chgrp -R 0 /opt/app-root \
    && chmod -R g+rwX /opt/app-root
USER 1001

EXPOSE 8080

ENTRYPOINT ["/opt/app-root/entrypoint.sh"]
```

**Important:** remove the old `CMD ["nginx", "-g", "daemon off;"]` because the entrypoint will start nginx.

---

## 3) One-time Angular changes you MUST have

### A) Ensure the template exists in your dist output

Create:

**`src/assets/env.template.js`**

```js
window.__env = {
  DEEPHAVEN_PSK: "__DEEPHAVEN_PSK__"
};
```

When you run `npm run build`, it will end up in:

```
dist/.../assets/env.template.js
```

### B) Load `env.js` in `src/index.html`

Add this before Angular scripts:

```html
<script src="assets/env.js"></script>
```

---

## 4) Build and run

### Build image

```bash
docker build -t thales-edge-ui:latest .
```

### Run with PSK

```bash
docker run -d \
  -p 4200:8080 \
  --name thales-edge-ui \
  -e DEEPHAVEN_PSK=79s9pkxpmri0 \
  thales-edge-ui:latest
```

---

## 5) Verify it worked (this will save you time)

### Check the template exists inside container

```bash
docker exec -it thales-edge-ui sh -c "ls -ltra /opt/app-root/src/assets | head"
```

### Check `env.js` got generated

```bash
docker exec -it thales-edge-ui sh -c "cat /opt/app-root/src/assets/env.js"
```

### Check from host

```bash
curl -s http://localhost:4200/assets/env.js
```

---

## Do you need to run `entrypoint.sh` manually each time PSK changes?

✅ **No. Never manually.**

- `entrypoint.sh` runs **automatically** every time the container starts.
    
- If PSK changes, just recreate the container:
    

```bash
docker rm -f thales-edge-ui

docker run -d \
  -p 4200:8080 \
  --name thales-edge-ui \
  -e DEEPHAVEN_PSK=NEW_PSK_VALUE \
  thales-edge-ui:latest
```

No rebuild needed for PSK changes.

---

If you paste the output of:

```bash
docker exec -it thales-edge-ui sh -c "ls -ltra /opt/app-root/src/assets; echo '---'; cat /opt/app-root/src/index.html | head -50"
```

…I can confirm in one shot whether (1) the template is present and (2) `index.html` is loading `assets/env.js`.