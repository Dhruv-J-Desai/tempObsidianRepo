Perfect — this screenshot proves **your font files are being served correctly**:

- `curl -I http://localhost:8080/assets/fonts/...woff2` → **HTTP/1.1 200 OK**
    
- `Content-Type: font/woff2`
    
- and the files exist in `/opt/app-root/src/assets/fonts`
    

✅ So the problem is **not** “fonts missing in the container”.

Now there are only 2 likely causes:

## 1) Your CSS bundle isn’t actually loading (most common)

When CSS isn’t loaded, the UI looks “messed up” exactly like your screenshot.

Run these (from host or inside container):

### Find the styles file name referenced by index.html

```bash
docker exec -it thales-edge sh -lc "grep -oE 'styles-[^\" ]+\\.css' /opt/app-root/src/index.html"
```

Suppose it prints: `styles-OLAOSOVV.css`

### Check nginx is serving that CSS as CSS (NOT index.html)

```bash
docker exec -it thales-edge sh -lc "curl -I http://localhost:8080/styles-OLAOSOVV.css"
docker exec -it thales-edge sh -lc "curl -s http://localhost:8080/styles-OLAOSOVV.css | head -n 5"
```

**What you want:**

- `Content-Type: text/css`
    
- body starts with CSS, **not** `<!doctype html>`
    

If you see HTML, your SPA fallback is accidentally serving `index.html` for CSS/JS.

### Fix (nginx): add a static-files block before the SPA fallback

```nginx
# serve real static files, otherwise 404 (do NOT fall back to index.html)
location ~* \.(css|js|map|woff2?|ttf|eot|svg|png|jpg|jpeg|gif|ico)$ {
  try_files $uri =404;
  access_log off;
}
```

Keep your SPA fallback after:

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

---

## 2) The fonts are available, but the CSS never _references_ them

Let’s confirm your built CSS actually contains the `@font-face` rules:

```bash
docker exec -it thales-edge sh -lc "grep -R \"@font-face\" -n /opt/app-root/src/styles-*.css | head"
docker exec -it thales-edge sh -lc "grep -R \"assets/fonts\" -n /opt/app-root/src/styles-*.css | head"
```

If these return nothing, then your `styles.scss` changes aren’t getting into the built output you copied into the image.

---

## One more “gotcha”: browser cache + same tag

If you rebuilt the image but kept the same tag (like `latest`), your browser might still cache the old `styles-XXXX.css`.

Do:

- hard refresh (Ctrl+Shift+R), or open in incognito
    
- and recreate container:
    

```bash
docker compose up -d --build --force-recreate
```

---

If you paste the output of:

1. the `grep styles-*.css` from `index.html`
    
2. the `curl -I` for that styles file  
    …I’ll tell you immediately whether it’s nginx fallback or build/copy/caching.