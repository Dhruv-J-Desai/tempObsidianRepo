Perfect — your **nginx.conf screenshot confirms the root cause**.

Your config says:

```nginx
location /jsapi/ {
  proxy_pass http://deephaven-local:10000/jsapi/;
  proxy_set_header Host $host;
}
```

But your **actual Deephaven container name** (from your earlier `docker ps`) is **`deephaven-local`**?  
Wait — in the screenshot it’s `deephaven-local` **with a hyphen**, but earlier you were using names like `deephaven-local` / `deephaven` / `deephaven-local:1.3.0`. If the container name is _not exactly_ `deephaven-local`, nginx will try to resolve a hostname that doesn’t exist and you’ll get exactly what you see: **ERR_EMPTY_RESPONSE**.

### ✅ Fix: point nginx to the _real_ container name on `dh-net`

Run this and look at the **NAMES** column:

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
```

Whatever name you see for Deephaven (example: `deephaven` or `deephaven-local`), use that in nginx.

For example, if your Deephaven container name is `deephaven` then nginx must be:

```nginx
location /jsapi/ {
  proxy_pass http://deephaven:10000/jsapi/;
  proxy_set_header Host $host;
}

location /grpc-web/ {
  proxy_pass http://deephaven:10000/grpc-web/;
  proxy_set_header Host $host;
}

location /dh/ {
  proxy_pass http://deephaven:10000/dh/;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_set_header Host $host;
}
```

---

## ✅ Also: you are missing one proxy route your UI needs

Your browser errors earlier included:

`/io.deephaven.proto.backplane.grpc.ConfigService/...`

That is **NOT** `/grpc-web/` and **NOT** `/dh/`.

So add this too:

```nginx
location /io.deephaven.proto. {
  proxy_pass http://<DEEPhaven_container_name>:10000;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_set_header Host $host;
}
```

Replace `<DEEPhaven_container_name>` with the real name.

---

## ✅ Quick verification (this will tell us instantly)

Run these commands:

### 1) From inside thales-edge, can it reach Deephaven?

```bash
docker exec -it thales-edge sh -lc "getent hosts deephaven-local || true; getent hosts deephaven || true"
```

### 2) Can thales-edge fetch dh-core.js through nginx proxy?

```bash
curl -I http://localhost:4200/jsapi/dh-core.js
```

If #2 returns `200` (or `304`), your Angular page will stop showing ERR_EMPTY_RESPONSE.

---

## Why “Angular is working but Deephaven isn’t”

Because `/` and your UI assets are served locally by nginx (`root /opt/app-root/src`), so UI loads fine.  
Only Deephaven calls (`/jsapi`, `/grpc-web`, `/io.deephaven.proto`, `/dh`) depend on correct nginx upstream.

---

### Do this change and you’re done

1. Update hostname in nginx.conf to match Deephaven container name exactly
    
2. Add the missing `location /io.deephaven.proto.` block
    
3. Rebuild/restart thales-edge image/container
    

If you paste the output of:

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
```

I’ll tell you the **exact final nginx.conf** with the correct name filled in.