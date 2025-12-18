- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Yep — this new screenshot shows **why your `npm install @angular/cdk@16.2.14` still fails**:

✅ You _think_ you’re on Angular 16…  
❌ but your project is currently still pinned to **Angular 20**:

```
Found: @angular/common@20.3.0
"@angular/common@20.3.0" from the root project
```

So right now your `package.json` (or an existing lock/partial install) still has:

- `@angular/common: 20.3.0` (and likely other `@angular/*` at 20.x)
    

That’s why CDK 16 refuses — it expects Angular 16/17.

---

# Fix (do it in the right order)

## 1) Hard reset (again) — because the repo is in a mixed state

Run this from the project root:

```bash
rm -rf node_modules package-lock.json .angular
```

(You already did earlier, but do it again now because installs have partially happened.)

---

## 2) Force Angular 16 FIRST (core packages)

Run:

```bash
npm install \
  @angular/core@16.2.16 \
  @angular/common@16.2.16 \
  @angular/compiler@16.2.16 \
  @angular/platform-browser@16.2.16 \
  @angular/platform-browser-dynamic@16.2.16 \
  @angular/router@16.2.16 \
  @angular/forms@16.2.16 \
  @angular/animations@16.2.16
```

This step is critical: it removes the `20.3.0` pins.

---

## 3) Then install Angular CLI + build tooling (still 16)

```bash
npm install -D \
  @angular/cli@16.2.16 \
  @angular/compiler-cli@16.2.16 \
  @angular-devkit/build-angular@16.2.16 \
  typescript@5.1.6
```

---

## 4) Then install CDK/Material for Angular 16

```bash
npm install @angular/cdk@16.2.14
# if you use material:
npm install @angular/material@16.2.14
```

---

## 5) Finally run a clean install to lock everything

```bash
rm -rf node_modules package-lock.json
npm install
```

---

# Most likely you have these lines in package.json (you must change them)

Search in `package.json` for any `@angular/*` showing 20.x:

```bash
cat package.json | grep -E "\"@angular/" -n
```

If you see stuff like:

```json
"@angular/common": "20.3.0"
"@angular/core": "20.3.0"
"@angular/cdk": "^20.2.7"
```

Change all of them to **16.2.x**.

---

## Why your earlier “big install” didn’t stick

Because you probably still have **some `@angular/*` pinned to 20.x in package.json**, and npm will honor the root project pins first.

---

If you paste just your `package.json` `dependencies` section (only the `@angular/*` lines are enough), I’ll tell you the exact edits so it’s consistent in one shot.