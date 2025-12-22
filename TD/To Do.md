Alright — here’s the **complete, end-to-end fix** so your **Angular (nginx container)** can load Deephaven JS (`/jsapi/dh-core.js`) and connect to Deephaven **without using `deephaven-local` from the browser**.

Right now your browser console shows:

- `GET http://localhost:4200/jsapi/dh-core.js` → **404**
    
- fallback `http://deephaven-local:10000/jsapi/dh-core.js` → **ERR_NAME_NOT_RESOLVED**
    

That’s expected because:

- `deephaven-local` is a **Docker hostname**, Windows browser can’t resolve it.
    
- Your **nginx config currently doesn’t proxy** `/jsapi` and `/grpc-web`, so it returns 404.
    

✅ Fix = **proxy Deephaven through nginx** and **in Angular always load dh-core via same-origin** (`/jsapi/...`), not via `DEEPhaven-local`.

---

# ✅ Target Architecture

Browser (Windows)  
→ `http://localhost:4200/...` (nginx in container)  
→ nginx proxies:

- `/jsapi/*` → Deephaven container `http://deephaven-local:10000/jsapi/*`
    
- `/grpc-web/*` → Deephaven container `http://deephaven-local:10000/grpc-web/*`
    
- `/dh/*` (WebSocket) → Deephaven container `http://deephaven-local:10000/dh/*`
    

So the browser never talks to `deephaven-local` directly.

---

# 1) Fix Angular code (most important)

### ✅ In `environment.ts` (or env.template.js if runtime)

Set base URL to **same origin**:

```ts
export const environment = {
  production: false,
  DEEPHAVEN_BASE_URL: '',     // IMPORTANT: empty => same-origin
  DEEPHAVEN_PSK: '...'
};
```

### ✅ In your DeephavenService use **same-origin** CoreClient URL

Instead of:

```ts
new this.dh.CoreClient(environment.DEEPHAVEN_BASE_URL)
```

Use:

```ts
new this.dh.CoreClient(window.location.origin)
```

### ✅ And load dh-core.js only from `/jsapi/dh-core.js` (not absolute)

Keep it simple:

```ts
private async ensureDhLoaded(): Promise<void> {
  if (this.dh) return;
  this.dh = (await import(/* @vite-ignore */ '/jsapi/dh-core.js')).default;
}
```

That’s it. **No fallback to deephaven-local**.

---

# 2) Add nginx reverse proxy (mandatory for docker build)

You are using `registry.access.redhat.com/ubi8/nginx-120` and copying dist to `/opt/app-root/src/`.

So you must provide nginx config file.

## Create `nginx.conf` in your repo (example)

```nginx
worker_processes  1;

events { worker_connections  1024; }

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;
  sendfile      on;

  server {
    listen 8080;
    server_name _;

    root /opt/app-root/src;
    index index.html;

    # Angular SPA fallback
    location / {
      try_files $uri $uri/ /index.html;
    }

    # Deephaven JS API
    location /jsapi/ {
      proxy_pass http://deephaven-local:10000/jsapi/;
      proxy_set_header Host $host;
    }

    # Deephaven gRPC-web
    location /grpc-web/ {
      proxy_pass http://deephaven-local:10000/grpc-web/;
      proxy_set_header Host $host;
    }

    # Deephaven sessions/IDE websocket
    location /dh/ {
      proxy_pass http://deephaven-local:10000/dh/;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
    }
  }
}
```

---

# 3) Update Dockerfile (copy nginx.conf)

Add these lines in your **runtime stage**:

```dockerfile
# Copy nginx config
COPY nginx.conf /etc/nginx/nginx.conf
```

So runtime stage becomes:

```dockerfile
FROM registry.access.redhat.com/ubi8/nginx-120

COPY --from=build /app/dist/thales-edge/browser /opt/app-root/src/
COPY nginx.conf /etc/nginx/nginx.conf

COPY entrypoint.sh /opt/app-root/entrypoint.sh
...
EXPOSE 8080
ENTRYPOINT ["/opt/app-root/entrypoint.sh"]
```

---

# 4) Ensure containers are on same docker network

Create a network:

```bash
docker network create dh-net
```

Run Deephaven on it with name `deephaven-local`:

```bash
docker rm -f deephaven-local 2>/dev/null || true

docker run -d --name deephaven-local --network dh-net \
  -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSK=my-fixed-psk \
  deephaven-local:1.2.0
```

Run Angular/nginx on same network:

```bash
docker rm -f thales-edge 2>/dev/null || true

docker run -d --name thales-edge --network dh-net \
  -p 4200:8080 \
  -e DEEPHAVEN_PSK=my-fixed-psk \
  thales-edge:latest
```

✅ Now nginx can resolve `deephaven-local` and proxy to it.

---

# 5) Verify (these must work)

From Windows browser:

1. Angular app  
    `http://localhost:4200/`
    
2. Deephaven JS file proxied through nginx  
    `http://localhost:4200/jsapi/dh-core.js`  
    ✅ should download JS, NOT 404
    
3. Deephaven health-ish check via proxy  
    `http://localhost:4200/jsapi/`  
    ✅ should return something (not required pretty)
    

If #2 works, your dynamic import will work.

---

# 6) Why your current setup fails (in 1 line)

Because you’re building Angular with nginx but still relying on **Angular dev proxy + docker hostname in browser**, and nginx isn’t proxying `/jsapi`.

---

## If you want, paste your `entrypoint.sh`

Some OpenShift nginx images need the config placed in `/etc/nginx/conf.d/default.conf` instead of `/etc/nginx/nginx.conf`. If your current container ignores my config, I’ll adjust it to the exact UBI nginx layout you’re using.