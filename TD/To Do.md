- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Perfect — you already have the **Node.js tarball (`node-v20.11.0-linux-x64.tar.xz`) downloaded locally**, and Docker Hub access is blocked.  
Now let’s **use that tarball directly in your Dockerfile**, replacing the `FROM … nodejs` image completely.

I’ll show you **exactly how to adapt your existing multi-stage Angular build Dockerfile**.

---

## ✅ What you have (confirmed from your screenshot)

- Node tarball present locally:
    
    ```
    node-v20.11.0-linux-x64.tar.xz
    ```
    
- Angular project with:
    
    ```
    package.json
    package-lock.json
    dist/thales-edge/browser
    ```
    
- Corporate npm registry:
    
    ```
    https://repo.td.com/repository/tds-npm-all-release/
    ```
    

---

## 🎯 Goal

Replace this:

```dockerfile
FROM snapshots.dev.registry.td.com/ubi8/nodejs-18:1-71 AS build
```

With:

- A **minimal base OS**
    
- **Manually installed Node from tarball**
    
- Same `npm ci` + `npm run build`
    

---

## ✅ Recommended Dockerfile (Node tarball–based, Docker Hub free)

### 📄 Dockerfile

```dockerfile
############################################
# Stage 1: Build Angular app using Node tarball
############################################
FROM registry.access.redhat.com/ubi8/ubi-minimal AS build

WORKDIR /app

# Install required OS tools
RUN microdnf install -y \
      tar \
      xz \
      gzip \
      shadow-utils \
    && microdnf clean all

# Copy Node tarball into image
COPY node-v20.11.0-linux-x64.tar.xz /opt/

# Install Node
RUN mkdir -p /opt/node \
    && tar -xJf /opt/node-v20.11.0-linux-x64.tar.xz -C /opt/node --strip-components=1 \
    && ln -s /opt/node/bin/node /usr/local/bin/node \
    && ln -s /opt/node/bin/npm /usr/local/bin/npm \
    && ln -s /opt/node/bin/npx /usr/local/bin/npx \
    && rm /opt/node-v20.11.0-linux-x64.tar.xz

# Verify node installation
RUN node --version && npm --version

# Copy dependency files first (layer caching)
COPY package.json package-lock.json ./

# Configure corporate npm registry and install deps
RUN npm config set registry https://repo.td.com/repository/tds-npm-all-release/ \
    && npm ci

# Copy rest of the source
COPY . .

# Build Angular
RUN npm run build


############################################
# Stage 2: Runtime image (Nginx)
############################################
FROM registry.access.redhat.com/ubi8/nginx-120

COPY --from=build /app/dist/thales-edge/browser /usr/share/nginx/html/

EXPOSE 80
```

---

## 📁 Directory layout (important)

Your build context **must include the tarball**:

```
thales-edge/
├── Dockerfile
├── node-v20.11.0-linux-x64.tar.xz
├── package.json
├── package-lock.json
├── angular.json
├── src/
└── dist/
```

---

## 🚀 Build command

From inside `thales-edge`:

```bash
docker build -t thales-edge-ui .
```

---

## 🔍 Why this works in your environment

✔ No Docker Hub  
✔ No `node:` image  
✔ Works with **air-gapped / restricted networks**  
✔ Uses **official Node binaries**  
✔ Matches your existing Angular pipeline

---

## ⚠️ Common pitfalls (watch out)

1. **Wrong tarball architecture**
    
    - Use `linux-x64` for UBI8
        
    - NOT `darwin` or `arm64` unless required
        
2. **Missing `xz`**
    
    - `.tar.xz` requires `xz` (already handled above)
        
3. **Angular dist path**
    
    - Must match:
        
        ```
        dist/thales-edge/browser
        ```
        

---

## 🔁 Optional: downgrade to Node 18 (matches your old image)

Just replace:

```dockerfile
COPY node-v20.11.0-linux-x64.tar.xz /opt/
```

With:

```dockerfile
COPY node-v18.16.1-linux-x64.tar.xz /opt/
```

And adjust the filename in the `tar` command.

---

## 🧠 If you want next

I can help you with:

- **Slimmer runtime image**
    
- **Offline npm cache**
    
- **Podman compatibility (WEnix)**
    
- **Multi-arch (x64 + ARM)**
    
- **Private Nexus-hosted Node tarball**
    

Just tell me 👍