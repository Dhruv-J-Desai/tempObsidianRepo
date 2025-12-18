- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

I see the problem now from your screenshots:

✅ You fixed most `@angular/*` to **16.2.16** and CDK/Material to **16.2.14**  
❌ But your dependency set is still **not a valid Angular 16 set**, mainly because:

1. **`zone.js` is still `0.15.0`** (that’s for newer Angular). Angular 16 expects `zone.js` **~0.13.x**.
    
2. You’re missing a couple of core Angular packages that are normally present (`platform-browser-dynamic`, `animations`), which can confuse resolution.
    
3. Your npm error shows `Found: @angular/common@undefined` — that usually happens when the install is half-resolving + the tree is inconsistent. Fixing the versions + doing a clean install resolves it.
    

Do this exactly:

---

## 1) Fix `package.json` versions (copy these)

### `dependencies`

Make sure you have **all** of these:

```json
"dependencies": {
  "@angular/animations": "16.2.16",
  "@angular/common": "16.2.16",
  "@angular/compiler": "16.2.16",
  "@angular/core": "16.2.16",
  "@angular/forms": "16.2.16",
  "@angular/platform-browser": "16.2.16",
  "@angular/platform-browser-dynamic": "16.2.16",
  "@angular/router": "16.2.16",

  "@angular/cdk": "16.2.14",
  "@angular/material": "16.2.14",

  "rxjs": "7.8.1",
  "tslib": "2.3.0",
  "zone.js": "0.13.3"
}
```

✅ Change your current `zone.js: 0.15.0` → **0.13.3** (this is important)

Keep your Deephaven + ag-grid deps as they are.

### `devDependencies`

Make sure these exist:

```json
"devDependencies": {
  "@angular/cli": "16.2.16",
  "@angular/compiler-cli": "16.2.16",
  "@angular-devkit/build-angular": "16.2.16",
  "typescript": "5.1.6"
}
```

🚫 Also ensure you **do NOT** have:

- `@angular/build` (delete it if present)
    

---

## 2) Clean EVERYTHING and reinstall

Run:

```bash
rm -rf node_modules package-lock.json
npm cache clean --force
npm install
```

---

## 3) Sanity check

```bash
npm ls @angular/core @angular/common @angular/cdk zone.js
```

You should see:

- `@angular/*` → 16.2.x
    
- `@angular/cdk` → 16.2.14
    
- `zone.js` → 0.13.3
    

---

### If it STILL fails

Run one command and paste the output:

```bash
npm ls @angular/common @angular/cdk @angular/material zone.js --all
```

But 99% of the time, **fixing zone.js + removing @angular/build + clean install** resolves this loop.

If you want, paste your full `dependencies` + `devDependencies` (just those blocks) and I’ll point out the last mismatched line.