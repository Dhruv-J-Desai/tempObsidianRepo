- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

You’re **very close**. That screenshot shows **two different issues**:

## 1) PowerShell error about `package-lock.json` not found

That’s fine — it just means you **don’t have a `package-lock.json`** in this repo. Ignore it.

## 2) You ran the flag wrong once

You typed:

- `npm install ==legacy-peer-deps` ❌ (double `==` makes npm think it’s a package name)
    

Correct is:

- `npm install --legacy-peer-deps` ✅
    

(You did run the correct one after, so good.)

## 3) Real blocker: `ETARGET No matching version found for @angular/animations@16.2.16`

This is almost always because **your registry/proxy doesn’t have that exact package version** (or it’s pointing to the wrong repo).

From your earlier screenshot, your registry is:

- `https://repo.td.com/repository/3rd-party-npm-central`
    

That repo might not host **all** Angular patch versions.

### Fix: point this project to the “all” npm repo (the one that contains everything)

Create a **project-local** `.npmrc` in the repo root (`thales-edge/.npmrc`) with:

```ini
registry=https://repo.td.com/repository/tds-npm-all-release/
always-auth=true
strict-ssl=false
```

Then run:

```powershell
npm config get registry
```

Make sure it prints `tds-npm-all-release` (not `3rd-party-npm-central`).

Now clean + install:

```powershell
Remove-Item -Recurse -Force node_modules -ErrorAction SilentlyContinue
Remove-Item -Force package-lock.json -ErrorAction SilentlyContinue
npm cache clean --force
npm install --legacy-peer-deps
```

---

## If you still get `No matching version found`

Then your Nexus repo truly doesn’t have `16.2.16`.

Two ways out:

### Option A (best): use an Angular patch version that **exists** in your Nexus

Try installing `16.2.0` or `16.2.14` everywhere (keep all Angular packages same patch):

In `package.json`, set all `@angular/*` to **16.2.14** (or one patch you know exists), including:

- `@angular/animations`, `common`, `compiler`, `core`, `forms`, `platform-browser`, `platform-browser-dynamic`, `router`
    
- dev deps: `@angular/cli`, `@angular/compiler-cli`, `@angular-devkit/build-angular`
    

Then:

```powershell
npm install --legacy-peer-deps
```

### Option B: temporarily use public npmjs (if policy allows)

Only if permitted in your environment.

---

### One quick check (to confirm repo has that version)

Run:

```powershell
npm view @angular/animations versions --json
```

If your registry is correct, this will return a big list. If `16.2.16` is not in that list, your Nexus doesn’t have it.

If you paste the output of:

- `npm config get registry`
    
- `npm view @angular/animations version`  
    I can tell you immediately whether it’s a **registry** issue or a **package version availability** issue.






Put those lines in a **project-level `.npmrc`** file in the **root of your Angular repo** (same folder where `package.json` is).

### Where

Example (from your screenshot):  
`C:\Users\TAP3507\source\projects\TDSBI-POC\BIShowcase2025\thales-edge\`

So create:  
`C:\Users\TAP3507\source\projects\TDSBI-POC\BIShowcase2025\thales-edge\.npmrc`

### How (Windows PowerShell)

Run this **inside the repo root**:

```powershell
cd C:\Users\TAP3507\source\projects\TDSBI-POC\BIShowcase2025\thales-edge

@"
registry=https://repo.td.com/repository/tds-npm-all-release/
always-auth=true
strict-ssl=false
"@ | Set-Content -Encoding ascii .npmrc
```

Verify:

```powershell
Get-Content .npmrc
npm config get registry
```

Then clean + install:

```powershell
Remove-Item -Recurse -Force node_modules -ErrorAction SilentlyContinue
Remove-Item -Force package-lock.json -ErrorAction SilentlyContinue
npm cache clean --force
npm install --legacy-peer-deps
```

(If you’re using Git Bash instead of PowerShell, tell me and I’ll give the exact commands for that shell.)