Nice — this screenshot confirms **CSS is loading correctly** ✅

- `curl -I http://localhost:8080/styles-OLAOSOVV.css` → **HTTP/1.1 200 OK**
    
- `Content-Type: text/css`
    
- And the body you printed is real CSS (not `index.html`)
    

So the “messed up” UI is **not** because nginx is serving HTML for CSS/JS.

That leaves 2 common causes:

---

## A) The font files are served, but the browser is **blocking** them (CORS / MIME / caching)

Even though `curl -I` shows `Content-Type: font/woff2`, browsers can still block if headers are missing.

Check headers for a font URL:

```bash
docker exec -it thales-edge sh -lc "curl -I http://localhost:8080/assets/fonts/flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2"
```

If you **don’t** see `Access-Control-Allow-Origin`, add this to nginx:

```nginx
location ~* \.(woff2?|ttf|eot|otf)$ {
  add_header Access-Control-Allow-Origin "*" always;
  add_header Cache-Control "public, max-age=31536000, immutable";
  try_files $uri =404;
  access_log off;
}
```

(Fonts and CSS are same origin here, but some setups still hit CORS edge cases—this removes doubt.)

---

## B) Your `@font-face` `src:` URLs in CSS don’t match the built output

In Angular builds, the CSS often gets rewritten and sometimes paths change depending on `<base href>` and deploy URL.

### 1) Confirm the CSS references the exact same path you’re curling:

```bash
docker exec -it thales-edge sh -lc "grep -R \"assets/fonts\" -n /opt/app-root/src/styles-*.css | head -n 20"
```

You want to see:  
`url(/assets/fonts/<file>.woff2)` ✅

If it shows something like:

- `url(assets/fonts/...)` (missing leading slash)
    
- `url(/opt/app-root/src/assets/fonts/...)` (wrong)
    
- `url(./assets/fonts/...)` (can break depending on base href)
    

**Best practice for SPA behind nginx root**:

```css
src: url("/assets/fonts/xxxx.woff2") format("woff2");
```

---

## Quick browser-side proof (fastest check)

Open DevTools → **Network** tab → filter `woff2`:

- If you see **404** → path mismatch (fix CSS path)
    
- If you see **200** but **(blocked)** or **CORS** → add nginx headers
    
- If you see fonts load but still “messed up” → it’s **not fonts**, it’s layout CSS (likely Angular Material theme vars, or missing global styles)
    

---

## One more thing: the “messed up” screenshot looks like “Material styles not applied”

That happens when **theme variables** aren’t taking effect or the material CSS isn’t included the way you think.

Confirm Material variables exist in the loaded CSS:

```bash
docker exec -it thales-edge sh -lc "grep -R \"--mat-sys\" -n /opt/app-root/src/styles-*.css | head"
```

If this returns nothing, your theme build/config isn’t being included.

---

### Send me just 2 outputs and I’ll tell you exactly which branch it is:

```bash
docker exec -it thales-edge sh -lc "curl -I http://localhost:8080/assets/fonts/flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2"
```

```bash
docker exec -it thales-edge sh -lc "grep -R \"assets/fonts\" -n /opt/app-root/src/styles-*.css | head -n 10"
```