- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Got it — if you must stay on **Node v18.16.1**, you need to **downgrade Angular (and its build tooling)** to a version that _supports_ Node 18.16.

From your error, your current Angular CLI is new enough that it requires **Node ≥ 20.19**, so you need to go back to an Angular major that supports Node 18.16 reliably.

## The safest target for Node 18.16

**Angular 16.2.x** is the most practical “works on Node 18.x” target (Angular 17+ often pushes Node requirements higher and has bigger tooling changes).

---

# Step-by-step downgrade (works even if `ng` won’t run)

### 0) Create a branch

```bash
git checkout -b downgrade-angular-node18
```

### 1) Downgrade Angular packages to 16.2.x

Run this from the Angular project root:

```bash
npm i -D @angular/cli@16.2.16 @angular/compiler-cli@16.2.16 typescript@5.1.6

npm i @angular/core@16.2.16 @angular/common@16.2.16 @angular/compiler@16.2.16 \
      @angular/platform-browser@16.2.16 @angular/platform-browser-dynamic@16.2.16 \
      @angular/router@16.2.16 rxjs@7.8.1 zone.js@0.13.3
```

If you use Angular forms/animations/http, also pin them:

```bash
npm i @angular/forms@16.2.16 @angular/animations@16.2.16 @angular/common@16.2.16
```

### 2) Pin the Angular build tooling

```bash
npm i -D @angular-devkit/build-angular@16.2.16 @angular-devkit/core@16.2.16 @angular-devkit/schematics@16.2.16
```

### 3) Remove/adjust Vite-related deps if you have them

Your earlier logs show `vite@7.x` etc. If your `package.json` has `vite`, `@vitejs/*`, or newer `yargs@18`, they’ll keep dragging Node 20 requirements.

Remove Vite if it’s a direct dependency:

```bash
npm remove vite
```

(If it’s coming transitively, the Angular 16 tooling usually avoids it.)

### 4) Clean install

```bash
rm -rf node_modules package-lock.json
npm install
```

### 5) Verify Node compatibility + build

```bash
node -v   # should be v18.16.1
npx ng version
npx ng build
```

---

# If your project is Angular 17/18/19 features (possible issues)

Downgrading to 16 can break if you rely on newer features like:

- `application` builder, newer SSR/prerender layout (`dist/<app>/browser` + `prerendered-routes.json`)
    
- standalone bootstrap patterns introduced/changed later
    

If `ng build` complains about builders / `angular.json`, tell me the exact error and I’ll give you the exact edits (usually it’s switching builders back to the classic `@angular-devkit/build-angular:browser` setup).

---

## One quick check (to confirm what you have)

Paste these sections from your `package.json`:

- `dependencies`
    
- `devDependencies`
    
- `scripts`
    

…and I’ll tell you **exactly which packages are forcing Node 20**, and the **minimum downgrade set** for your specific app (instead of a broad downgrade).