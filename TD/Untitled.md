Got it — and **this is a super common symptom**: the app loads, but **CSS/fonts aren’t being applied**, so everything looks “unstyled / messed up”.

From your screenshot I can see fonts **do exist** inside the container:

`/opt/app-root/src/assets/fonts/...woff2`

So the problem is not “fonts missing”. It’s almost always one of these:

## 1) The font files are NOT being served (404) even though they exist

Check from inside the container (this is the fastest proof):

```bash
docker exec -it thales-edge sh -lc "curl -I http://localhost:8080/assets/fonts/Roboto/Roboto-Regular.woff2"
```

Expected:

- `HTTP/1.1 200 OK`
    

If you get `404` → your nginx root or path routing is wrong.

Also check in browser DevTools → **Network** → filter `woff2` → see if it’s `200` or `404`.

---

## 2) The CSS that defines `@font-face` is not loading (or loading from a wrong path)

Check if your built CSS actually contains “Roboto”:

```bash
docker exec -it thales-edge sh -lc "grep -R \"font-face\" -n /opt/app-root/src | head"
docker exec -it thales-edge sh -lc "grep -R \"Roboto\" -n /opt/app-root/src | head"
```

If these return nothing → your `styles.scss` isn’t being included in the build you’re serving.

Then check what CSS file your index.html is pointing to:

```bash
docker exec -it thales-edge sh -lc "sed -n '1,120p' /opt/app-root/src/index.html | cat"
```

Look for something like:  
`styles-XXXXXXXX.css`

Then verify that file exists:

```bash
docker exec -it thales-edge sh -lc "ls -1 /opt/app-root/src/styles*"
```

If index.html references a CSS filename that doesn’t exist → you’re serving a mismatched build output.

---

## 3) Base href / deploy path mismatch (VERY common with nginx + Angular)

If your `index.html` has:

```html
<base href="/">
```

then assets are requested as:  
`/assets/...`

But if you’re serving the app under a subpath (like `/app-root/` or `/thales-edge/`) then the browser requests the wrong URLs and everything breaks.

✅ Quick proof: open DevTools Console/Network and see requests like:

- `/styles-xxx.css` 404
    
- `/assets/...` 404
    

### Fix options

- Serve from `/` and keep `<base href="/">`
    
- OR build with a base href:
    

```bash
ng build --base-href /thales-edge/ --deploy-url /thales-edge/
```

and make nginx serve it under that same path.

---

## 4) Font is loaded but not applied (CSS specificity / Material expects Roboto)

Make sure you actually **use** Roboto somewhere, e.g. in `styles.scss`:

```scss
body {
  font-family: "Roboto", Arial, sans-serif;
}
```

And ensure your `@font-face` names match exactly:

```scss
@font-face {
  font-family: "Roboto";
  src: url("/assets/fonts/Roboto/Roboto-Regular.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
}
```

⚠️ Note: use `/assets/...` (absolute) to avoid relative path issues.

---

# The most likely cause in your case

Since the UI looks “messed up”, it’s usually **not just fonts** — it’s **styles.css not loading** due to a **base href / path / nginx root mismatch**.

If you do just ONE thing now, do this:

### ✅ Check if the main CSS is being served

```bash
docker exec -it thales-edge sh -lc "ls -1 /opt/app-root/src | egrep 'styles|main|polyfills'"
```

Then in browser Network check:

- `styles-*.css` is **200**
    
- `assets/fonts/*.woff2` is **200**
    

---

If you paste the output of these two commands, I’ll tell you exactly which case it is:

```bash
docker exec -it thales-edge sh -lc "ls -1 /opt/app-root/src | head -50"
docker exec -it thales-edge sh -lc "sed -n '1,80p' /opt/app-root/src/index.html"
```