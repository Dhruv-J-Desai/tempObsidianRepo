I’m going to pinpoint _exactly_ why it’s still trying to connect to `ws://localhost:4200` (which means the proxy isn’t being used), then give you the precise fixes.

That screenshot is the key: **it’s still trying to call Deephaven gRPC endpoints at the ROOT path on your Angular origin**:

`ws://localhost:4200/io.deephaven.proto.backplane.grpc.ConfigService/GetConfigurationConstants`

✅ This means: **your proxy config is incomplete**.  
You proxied only:

- `/jsapi`
    
- `/grpc-web`
    

…but Deephaven’s JS client is actually calling:

- **`/io.deephaven.proto.*`** (this is the gRPC-web endpoint path)
    

So the browser hits Angular (4200) → Angular doesn’t know that path → disconnect loop.

---

# Fix (Option A / Angular dev server): update proxy.conf.json

Your current proxy has:

```json
"/jsapi": {...},
"/grpc-web": {...}
```

✅ Replace it with this (add the missing Deephaven gRPC paths):

```json
{
  "/jsapi": {
    "target": "http://localhost:10000",
    "changeOrigin": true,
    "secure": false,
    "ws": true
  },
  "/grpc-web": {
    "target": "http://localhost:10000",
    "changeOrigin": true,
    "secure": false,
    "ws": true
  },

  "/io.deephaven.proto.": {
    "target": "http://localhost:10000",
    "changeOrigin": true,
    "secure": false,
    "ws": true
  }
}
```

### Then restart Angular (important)

Stop `ng serve` and run again:

```bash
ng serve --proxy-config proxy.conf.json
```

---

# DeephavenService.ts (Option A): what it should be

✅ **If you are using Angular dev proxy**, then **YES** you should keep using `window.location.origin`:

```ts
this.client = new this.dh.CoreClient(window.location.origin);
```

Because Angular is acting as the reverse proxy for Deephaven.

So for Option A:

- **No need to hardcode `http://localhost:10000` in CoreClient**
    
- Just make sure proxy routes include `/io.deephaven.proto.`
    

---

# Quick verification (do this in browser)

After restart, open these in the browser:

1. ✅ should show JS file (not 404):
    

- `http://localhost:4200/jsapi/dh-core.js`
    

2. ✅ should NOT be 404 anymore (may show 405/400, that’s fine):
    

- `http://localhost:4200/io.deephaven.proto.backplane.grpc.ConfigService/GetConfigurationConstants`
    

If #2 stops being **404**, your proxy is correct.

---

# If you are running thales-edge in Docker (nginx) instead of ng serve

Then Angular proxy file is NOT used. In that case you must add an nginx rule too:

```nginx
location ^~ /io.deephaven.proto. {
  proxy_pass http://deephaven:10000;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
```

(And restart/rebuild the container.)

---

## Why you’re seeing `ws://localhost:4200/...`

Because the JS client connects to **the CoreClient base URL** (your Angular origin), and expects your proxy to forward all Deephaven API paths. Right now it forwards only `/jsapi` and `/grpc-web`, but Deephaven also uses `/io.deephaven.proto.*`.

---

If you want, send **one screenshot of your `ng serve` command** (or tell me if you’re using Docker nginx), and I’ll give you the exact final config for that one path only.