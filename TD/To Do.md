- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Great, this screenshot shows the **last blocking issue very clearly**.  
You’re _almost_ done with the downgrade — this is now a **version alignment problem**, not a conceptual one.

---

## What the error really says (decoded)

Key parts from the screenshot:

```
Could not resolve dependency:
dev @angular-devkit/build-angular@16.2.16
peer @angular/compiler-cli "^16.0.0"
```

But earlier in the same output:

```
Found: @angular/compiler-cli@16.2.16
node_modules/@angular/compiler-cli
```

So at first glance this looks fine… **but** npm is still failing because:

### 🔴 You still have a mismatched Angular dependency somewhere

Most commonly one of these is true:

1. ❌ Some `@angular/*` package is **not** at `16.2.16`
    
2. ❌ `typescript` version is incompatible
    
3. ❌ A leftover Vite / AnalogJS package is still pulling newer Angular peers
    
4. ❌ `package-lock.json` still contains newer Angular metadata
    

---

## The rule for Angular downgrades (important)

**ALL** Angular packages must be on the **same minor version**.

That means:

```
@angular/*            → 16.2.16 (all of them)
@angular-devkit/*     → 16.2.16
@angular/compiler-cli → 16.2.16
typescript            → 5.1.6
```

Even **one** straggler will break `npm install`.

---

## Do this EXACTLY (clean reset, no guessing)

### 1️⃣ Remove all build artifacts and lockfiles

```bash
rm -rf node_modules package-lock.json .angular
```

---

### 2️⃣ Force-install a fully aligned Angular 16 toolchain

Run this **as one command**:

```bash
npm install \
  @angular/cli@16.2.16 \
  @angular/compiler@16.2.16 \
  @angular/compiler-cli@16.2.16 \
  @angular/core@16.2.16 \
  @angular/common@16.2.16 \
  @angular/platform-browser@16.2.16 \
  @angular/platform-browser-dynamic@16.2.16 \
  @angular/router@16.2.16 \
  @angular/forms@16.2.16 \
  @angular/animations@16.2.16 \
  @angular-devkit/build-angular@16.2.16 \
  @angular-devkit/core@16.2.16 \
  @angular-devkit/schematics@16.2.16 \
  rxjs@7.8.1 \
  zone.js@0.13.3 \
  typescript@5.1.6
```

This **overwrites everything** consistently.

---

### 3️⃣ Remove any remaining Vite / AnalogJS (must be zero)

Run:

```bash
npm ls | grep -E "vite|analog"
```

If **anything** shows up, remove it:

```bash
npm remove @analogjs/vite-plugin-angular analogjs vite
```

Then repeat step 1 + 2.

---

### 4️⃣ Verify versions (before building)

```bash
npx ng version
```

You **must** see:

- Angular: **16.2.16**
    
- TypeScript: **5.1.6**
    
- Node: **18.16.1**
    

If yes → proceed.

---

### 5️⃣ Build

```bash
npx ng build
```

This **will work** once everything is aligned.

---

## Why this error kept looping

You were doing the right thing, but npm is strict:

- Angular 16 **does not tolerate mixed minors**
    
- Vite / AnalogJS **silently pulls Angular 18+ peers**
    
- One leftover dep poisons the tree
    

Now you’re fixing it the **correct, deterministic way**.

---

## After this

Once `ng build` works locally on Node 18, your **Dockerfile with `ubi8/nodejs-18` will work unchanged**.

If you want, next I can:

- Sanity-check your `package.json`
    
- Clean your `angular.json` (builders, SSR leftovers)
    
- Finalize the Dockerfile for Nexus + Node 18
    

You’re in the final stretch 👍