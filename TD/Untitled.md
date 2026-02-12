Yep — in a locked-down TD network, **Google Fonts is often blocked**, so those `<link href="https://fonts.googleapis.com/...">` will fail. You have a few solid workarounds.

## Option 1 (best): Bundle the font files with your app (no internet needed)

1. **Download the font files** once (outside TD network, or from an internal-approved source) as `.woff2` (preferred).
    
2. Put them in your UI project, e.g.
    

```
src/assets/fonts/Roboto/
  Roboto-Regular.woff2
  Roboto-Medium.woff2
```

3. Add `@font-face` in a CSS file (e.g. `src/styles.css` or `src/assets/fonts/fonts.css`):
    

```css
@font-face {
  font-family: "Roboto";
  src: url("/assets/fonts/Roboto/Roboto-Regular.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: "Roboto";
  src: url("/assets/fonts/Roboto/Roboto-Medium.woff2") format("woff2");
  font-weight: 500;
  font-style: normal;
  font-display: swap;
}
```

4. Use it normally:
    

```css
body { font-family: "Roboto", Arial, sans-serif; }
```

✅ Works everywhere, no external calls.

> Note: Make sure your org is okay with including the font license files in the repo (Google fonts are usually OFL/Apache, but still follow internal policy).

---

## Option 2: Use system fonts only (fastest, zero packaging)

If you don’t care about exact Roboto look:

```css
body {
  font-family: system-ui, -apple-system, "Segoe UI", Arial, sans-serif;
}
```

✅ No network, no extra files, always works.

---

## Option 3: Host fonts internally (if TD has an internal artifact/static host)

If you have an internal Nginx/static server or artifact repo:

- Upload `.woff2` there
    
- Point `<link>` or `@font-face` to the internal URL
    

✅ Centralized, no per-app duplication  
❗ Needs an approved internal hosting place

---

## Option 4: Package fonts into your Docker image (good for your “share image” goal)

If your UI is served by nginx in a container:

- Copy `assets/fonts` into the image at build time
    
- Reference `/assets/fonts/...` in CSS
    

✅ Teammates don’t need internet, just run the image

---

### Quick recommendation for your setup

Since you’re already containerizing (`thales-edge` / nginx):  
**Bundle fonts into the UI repo + serve them from your own nginx** (Option 1 + 4). That matches your “no external dependencies” goal.

If you tell me whether this is **Angular** (it looks like it from `<app-root>`) and whether the UI is built into nginx, I’ll give you the exact file paths + Dockerfile snippet for your repo layout.