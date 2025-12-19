- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Below is a complete, end-to-end **Option 1 (entrypoint.sh runtime injection)** setup for an **Angular app served by Nginx**.

And no — you **do not run `entrypoint.sh` manually**. Docker runs it **automatically on every container start**, so when you pass a new PSK via `docker run -e`, it regenerates `env.js` for that container.

---

# 1) Angular changes (one-time)

## A) Create a runtime template file

Create this file in your Angular project:

**`src/assets/env.template.js`**

```js
// This file is a template; it will be copied into the image.
// The container startup will generate env.js from Docker env vars.
window.__env = {
  DEEPHAVEN_PSK: "__DEEPHAVEN_PSK__"
};
```

## B) Load `env.js` in `index.html`

Edit:

**`src/index.html`**

Add this **before** your main scripts (usually before `</head>`):

```html
<script src="assets/env.js"></script>
```

> Keep it above the Angular bundle scripts so it’s available when Angular bootstraps.

## C) Read it in Angular code

Where you need the PSK:

```ts
const psk = (window as any).__env?.DEEPHAVEN_PSK;
console.log("PSK from runtime env:", psk);
```

(You can wrap it in a service if you want.)

---

# 2) entrypoint.sh (inside the Docker image)

Create:

**`entrypoint.sh`**

```sh
#!/bin/sh
set -e

# Generate runtime env.js from the template + Docker environment variable
# Writes into the nginx web root so the browser can fetch it at /assets/env.js

TEMPLATE="/usr/share/nginx/html/assets/env.template.js"
TARGET="/usr/share/nginx/html/assets/env.js"

if [ ! -f "$TEMPLATE" ]; then
  echo "ERROR: Missing template: $TEMPLATE"
  exit 1
fi

if [ -z "$DEEPHAVEN_PSK" ]; then
  echo "WARNING: DEEPHAVEN_PSK is empty; env.js will contain an empty value."
  DEEPHAVEN_PSK=""
fi

# Replace placeholder with runtime value
# Using sed (available in alpine) — no envsubst needed.
sed "s|__DEEPHAVEN_PSK__|${DEEPHAVEN_PSK}|g" "$TEMPLATE" > "$TARGET"

# Start nginx in foreground
exec nginx -g 'daemon off;'
```

---

# 3) Dockerfile (Angular + Nginx)

This assumes your Angular build output is `dist/thales-edge-ui` (adjust if different).

**`Dockerfile`**

```dockerfile
FROM nginx:alpine

# Copy built Angular files
COPY dist/thales-edge-ui/ /usr/share/nginx/html/

# Copy the runtime template into the served assets folder
COPY src/assets/env.template.js /usr/share/nginx/html/assets/env.template.js

# Copy entrypoint that will generate env.js at container startup
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

---

# 4) Build the image

From the project root:

```bash
# build Angular first (example)
npm ci
npm run build

# build docker image
docker build -t thales-edge-ui:latest .
```

---

# 5) Run with PSK (your main goal)

```bash
docker run -d \
  -p 4200:80 \
  --name thales-edge-ui \
  -e DEEPHAVEN_PSK=79s9pkxpmri0 \
  thales-edge-ui:latest
```

Now your Angular app can read it from:

- `window.__env.DEEPHAVEN_PSK`
    
- and the browser can fetch `assets/env.js`
    

---

# 6) Verify it worked (super useful)

### Check the generated file inside container

```bash
docker exec -it thales-edge-ui sh -c "cat /usr/share/nginx/html/assets/env.js"
```

### Or from browser / curl

```bash
curl -s http://localhost:4200/assets/env.js
```

You should see something like:

```js
window.__env = { DEEPHAVEN_PSK: "79s9pkxpmri0" };
```

---

# Do you need to run entrypoint.sh every time?

✅ **No manual step.**  
`entrypoint.sh` runs automatically **every time the container starts**.

## When PSK changes, what do you do?

You just **recreate the container** with the new env var:

```bash
docker rm -f thales-edge-ui

docker run -d \
  -p 4200:80 \
  --name thales-edge-ui \
  -e DEEPHAVEN_PSK=NEW_VALUE_HERE \
  thales-edge-ui:latest
```

That’s it. `entrypoint.sh` will generate a new `env.js` with the new PSK.

---

## One important warning (security)

This approach **exposes the PSK to anyone who can open the UI** (they can read `assets/env.js` or DevTools). If that PSK is sensitive, the safer approach is keeping it in the backend only.

---

If you tell me your actual Angular dist folder name (what shows under `dist/`), I’ll adjust the Dockerfile `COPY dist/...` line to match exactly so you can paste/run without edits.