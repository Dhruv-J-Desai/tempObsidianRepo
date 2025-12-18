- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Nice — this screenshot explains everything.

### What I see

- `npm config get registry` ✅ is **TD Nexus**:  
    `https://repo.td.com/repository/3rd-party-npm-central`
    
- But in your `npm config list -l` you also have:
    
    - `strict-ssl = false` (fine for corp, but not ideal)
        
    - `cafile = null`
        
    - and **no project `.npmrc`**
        

So installs are fully driven by **global/user config**, and if Nexus doesn’t have some packages/versions (or blocks optional deps), npm resolution can behave weirdly and you end up with `@angular/common@undefined` during dependency tree resolution.

---

## Do this: set a project-level `.npmrc` (so Docker + teammates are consistent)

Create a `.npmrc` in your repo root (same folder as `package.json`):

```bash
cat > .npmrc <<'EOF'
registry=https://repo.td.com/repository/3rd-party-npm-central/
always-auth=true
strict-ssl=false
EOF
```

Now confirm:

```bash
npm config get registry
npm config get userconfig
npm config get globalconfig
```

---

## Next: your real blocker right now is NOT registry — it’s Angular version mismatch

Your `package.json` shows Angular core packages at **16.2.16**, but earlier you had **CDK/Material 20.x** (or you temporarily installed it).  
That causes the exact error you’re seeing:

- `@angular/cdk@16.2.14` wants Angular `^16 || ^17`
    
- but your root had `@angular/common@20.3.0` at one point (your screenshot shows that conflict)
    

### Fix: force ALL Angular + CDK + Material to 16.2.x

Run this (copy-paste):

```bash
rm -rf node_modules package-lock.json
npm cache clean --force

npm i \
  @angular/animations@16.2.16 \
  @angular/common@16.2.16 \
  @angular/compiler@16.2.16 \
  @angular/core@16.2.16 \
  @angular/forms@16.2.16 \
  @angular/platform-browser@16.2.16 \
  @angular/platform-browser-dynamic@16.2.16 \
  @angular/router@16.2.16 \
  @angular/cli@16.2.16 \
  @angular/compiler-cli@16.2.16 \
  @angular-devkit/build-angular@16.2.16 \
  @angular/cdk@16.2.14 \
  @angular/material@16.2.14 \
  --save --save-exact
```

Then:

```bash
npm install --legacy-peer-deps
```

---

## About “downgrading to Node 18”

You already _are_ on Node 18 locally (`gular-node18` prompt).  
But remember: **Angular 20+ requires Node 20.19+**, so if you truly want to stay on Node 18, you must keep Angular around **16/17/18** (your current target is Angular 16, which is fine).

---

## If Docker build is the goal

Use a Node 18 base image in Docker, but **don’t use Angular 20** in that container.

Example:

```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps
COPY . .
RUN npm run build
```

---

If you paste your **full `dependencies` + `devDependencies`** blocks (text), I can tell you exactly which packages are still pulling Angular 20+ and give you the exact pin-set to make it clean.