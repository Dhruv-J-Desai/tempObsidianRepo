- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Got it — thanks for the screenshot. This error is **NOT about Node tarball anymore** 👍  
Your Node setup is actually working fine.

---

## 🔴 What is really failing?

From your log:

```
Inlining of fonts failed.
An error has occurred while retrieving
https://fonts.googleapis.com/css2?family=Roboto
SELF_SIGNED_CERT_IN_CHAIN
```

### ✅ Root cause

You are building Angular **inside Docker**, and during `ng build` Angular tries to:

➡️ Download Google Fonts  
➡️ Over HTTPS  
➡️ Docker container does **not trust your corporate SSL certificate**

So the build fails.

This is **very common in corporate/WEnix environments**.

---

## 🧠 Why this happens specifically during Angular build

Angular CLI does **font inlining** by default:

- It fetches Google Fonts
    
- Embeds them into the final CSS
    
- Requires outbound HTTPS access
    

Your Docker container:

- Has no corporate root CA
    
- Sees Google Fonts cert chain as invalid
    
- Fails with `SELF_SIGNED_CERT_IN_CHAIN`
    

---

## ✅ FIX OPTION 1 (Best & Cleanest): Disable font inlining

### Modify `angular.json`

Find:

```json
"build": {
  "options": {
```

Add:

```json
"inlineFonts": false
```

### Example:

```json
"options": {
  "outputPath": "dist/app",
  "index": "src/index.html",
  "main": "src/main.ts",
  "polyfills": ["zone.js"],
  "tsConfig": "tsconfig.app.json",
  "inlineFonts": false
}
```

✅ No HTTPS calls  
✅ No cert issues  
✅ Fastest fix  
✅ Recommended for Docker builds

---

## ✅ FIX OPTION 2: Use local fonts (Production-grade)

### Step 1: Download Roboto once

Download `.woff2` files (outside Docker).

### Step 2: Put in:

```
src/assets/fonts/
```

### Step 3: Use local CSS

```css
@font-face {
  font-family: 'Roboto';
  src: url('/assets/fonts/roboto.woff2') format('woff2');
}
```

Remove Google Fonts `<link>` from `index.html`.

✅ Fully offline  
✅ Enterprise-safe

---

## ⚠️ NOT Recommended but works (Temporary)

Disable SSL verification:

```dockerfile
ENV NODE_TLS_REJECT_UNAUTHORIZED=0
```

🚫 Security risk  
🚫 Should not go to production

---

## 🟢 Your Node tarball setup is OK

From your Dockerfile:

```dockerfile
COPY node-v20.19.0-linux-x64.tar.xz /opt/
RUN tar -xf ...
```

✅ Correct  
✅ Node is available  
✅ npm runs fine

The failure happens **later during Angular build**, not Node install.

---

## 🏁 Final Recommendation (What you should do)

✔ Disable `inlineFonts`  
✔ Rebuild Docker image  
✔ Keep Node tarball approach

If you want, paste:

- `angular.json`
    
- Dockerfile
    

I’ll give you a **fully hardened Dockerfile** for WEnix / corporate SSL environments.