You’re **very close** — your proxy config screenshot is correct now (it includes `/io.deephaven.proto.`).  
If you can open `http://localhost:10000/jsapi/dh-core.js` in Windows, Deephaven is reachable.

Now the remaining rule for **Option A (Angular dev proxy)** is:

✅ **CoreClient base must be your Angular origin**, NOT Deephaven’s base URL.  
Because the browser must go through the proxy (4200 → 10000).

So in `deephaven.service.ts`, set it like this.

---

## ✅ Correct `deephaven.service.ts` for Option A (Angular proxy)

### 1) Keep dynamic import through the proxy

```ts
const proxyUrl = '/jsapi/dh-core.js';
this.dh = (await import(/* @vite-ignore */ proxyUrl)).default;
```

### 2) CoreClient must use Angular origin (proxy host)

```ts
this.client = new this.dh.CoreClient(window.location.origin);
```

### 3) Login token (PSK) can still come from runtime env

```ts
const psk = (window as any).__env?.DEEPHAVEN_PSK;
if (psk) {
  await this.client.login({
    type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
    token: psk,
  });
}
```

---

## Full “known-good” `getClient()` block (copy/paste)

```ts
private async ensureDhLoaded(): Promise<void> {
  if (this.dh) return;
  const proxyUrl = '/jsapi/dh-core.js';
  this.dh = (await import(/* @vite-ignore */ proxyUrl)).default;
}

private async getClient(): Promise<any> {
  await this.ensureDhLoaded();
  if (this.client) return this.client;

  // IMPORTANT for Option A: use Angular origin so proxy is used
  this.client = new this.dh.CoreClient(window.location.origin);

  const psk = (window as any).__env?.DEEPHAVEN_PSK;
  if (psk) {
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });
  }

  return this.client;
}
```

✅ That’s it for Option A.

---

## Why your current code is risky

In your screenshot you have:

```ts
this.client = new this.dh.CoreClient(this.dhBaseUrl());
```

If `dhBaseUrl()` becomes `http://localhost:10000`, then the client will try to call gRPC endpoints directly from the browser to 10000 (CORS + ws issues), and you’ll see disconnect problems again.

**For Option A, don’t use `dhBaseUrl()` at all.**

---

## Quick “am I using the proxy?” test

Open DevTools → Network.

When the app runs, you should see requests like:

- `http://localhost:4200/io.deephaven.proto...`
    
- `http://localhost:4200/jsapi/...`
    

NOT:

- `http://localhost:10000/io.deephaven.proto...`
    

If you see `:4200`, proxy is being used correctly.

---

## One more important thing: restart Angular after proxy edits

Proxy changes don’t apply until you restart:

```bash
Ctrl+C
ng serve --proxy-config proxy.conf.json
```

---

If you send **one screenshot of your `angular.json` serve config** (or the exact `ng serve ...` command you’re running), I’ll tell you if the proxy file is actually being picked up.