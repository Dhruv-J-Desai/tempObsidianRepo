Perfect — this output is **exactly what we wanted** ✅

It proves:

- Your built CSS **does contain** the `@font-face` rules
    
- And they reference the fonts with the **correct URL**:
    

`src: url(/assets/fonts/<file>.woff2) format("woff2");`

So **the CSS path is correct**. Now the only question is: **is the browser actually downloading those `.woff2` files and using them?**

## Next 3 checks (in order)

### 1) Verify the font file is actually reachable + looks like a real woff2

Run:

```bash
docker exec -it thales-edge sh -lc "curl -I http://localhost:8080/assets/fonts/flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2"
docker exec -it thales-edge sh -lc "curl -s --range 0-15 http://localhost:8080/assets/fonts/flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2 | xxd"
```

Expected:

- `HTTP/1.1 200 OK`
    
- `Content-Type: font/woff2`
    
- The hex output should start with something like **`77 4f 46 32`** (that’s “wOF2”).
    

If instead it starts with `<html` or something, nginx is serving the wrong thing.

---

### 2) Check in the browser (this is the real decider)

Open DevTools → **Network** → filter `woff2` → refresh page.

You should see those exact font files:

- status **200**
    
- size **non-zero**
    
- **Type: font**
    

If you see:

- **404** → nginx root/path mismatch
    
- **(blocked)** / **CORS** → add nginx header (below)
    
- **200** but “from disk cache” and still old → hard reload / disable cache
    

Hard reload:

- Chrome DevTools open → right click refresh → **Empty cache and hard reload**
    

---

### 3) If fonts load but UI still “messed up”

Then it’s **not fonts**. That table/layout issue usually means **Material theme CSS / global styles** aren’t applying the way you think.

Quick check:

```bash
docker exec -it thales-edge sh -lc "grep -R \"--mat-sys\" -n /opt/app-root/src/styles-*.css | head"
```

If that prints nothing, your Material theme variables aren’t in the shipped CSS.

---

## If DevTools shows CORS blocking fonts (add this nginx block)

```nginx
location ~* \.(woff2?|ttf|otf|eot)$ {
  add_header Access-Control-Allow-Origin "*" always;
  add_header Cache-Control "public, max-age=31536000, immutable";
  try_files $uri =404;
}
```

---

If you paste the output of the **two curl commands** in step (1), I’ll tell you immediately whether nginx is serving the correct binary font (and what to fix if not).