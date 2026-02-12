Yes — **your path in `styles.scss` is correct** _for the way your app is being served_.

From your screenshot, the built files end up here:

- `dist/thales-edge/browser/assets/fonts/<file>.woff2`
    

And your nginx `root` is:

- `root /opt/app-root/src;`
    

So the browser URL that maps to that is:

- `/assets/fonts/<file>.woff2`
    

✅ That matches what you wrote:

```scss
src: url("/assets/fonts/flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2") format("woff2");
```

### But here’s the key detail:

In your screenshot the filenames look like:

- `flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2`
    
- `KFO7CnqEu92Fr1ME7kSn66aGLdTyIUAMa3GUBGEe.woff2`
    

Your CSS must match **exactly** (case + dashes). So do this:

```scss
@font-face {
  font-family: "Roboto";
  src: url("/assets/fonts/flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: "Roboto";
  src: url("/assets/fonts/KFO7CnqEu92Fr1ME7kSn66aGLdTyIUAMa3GUBGEe.woff2") format("woff2");
  font-weight: 500;
  font-style: normal;
  font-display: swap;
}
```

### Super important check (this will tell the truth instantly)

From inside the container, run:

```bash
docker exec -it thales-edge sh -lc 'curl -I http://localhost:8080/assets/fonts/flUhRq6tzZclQEJ-Vdg-IuiaDsNc.woff2'
```

If it returns **200**, path is correct.  
If it returns **404** or **200 with text/html**, nginx is still falling back to index.html (needs the `/assets/` block I sent earlier).

### One more improvement

Since you’re using nginx root `/opt/app-root/src`, make sure your container actually has:

`/opt/app-root/src/assets/fonts/...`

Run:

```bash
docker exec -it thales-edge sh -lc 'ls -l /opt/app-root/src/assets/fonts'
```

If your fonts are _instead_ inside something like `/opt/app-root/src/browser/assets/...`, then the URL path is correct but the **files are copied to the wrong folder** in the image.

If you paste the output of the two commands (`ls` and `curl -I`), I’ll tell you exactly which case it is and what to change (CSS vs Docker COPY vs nginx).