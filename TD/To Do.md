- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Nice, that screenshot actually shows exactly what we needed:

- Build succeeded ✅
    
- Output location:  
    `C:\Users\TAP3507\source\projects\TDSBI-POC\BIShowcase2025\thales-edge\dist\thales-edge`
    

That **`dist/thales-edge`** folder is your built Angular app.

The only problem now is:

```bash
bash: zip: command not found
```

Git Bash on Windows doesn’t have the `zip` program installed, so that command just isn’t available. No big deal; we’ll zip it using Windows tools instead.

---

## 1️⃣ Create the “jar-like” zip for the UI

### Option A – easiest (File Explorer)

1. Open Explorer to:
    
    ```text
    C:\Users\TAP3507\source\projects\TDSBI-POC\BIShowcase2025\thales-edge\dist\
    ```
    
2. Right-click the **`thales-edge`** folder  
    → **Send to ➜ Compressed (zipped) folder**
    
3. Rename the zip to something like:
    
    ```text
    ui-dist.zip
    ```
    
4. Move `ui-dist.zip` next to your shell scripts, e.g.:
    
    ```text
    C:\Users\TAP3507\source\shell-script-files\
        start-angular.sh
        ui-dist.zip
    ```
    

Now you have the **Angular equivalent of a JAR**: `ui-dist.zip`.

---

## 2️⃣ `start-angular.sh` that serves this zip

Put this in `shell-script-files/start-angular.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ARCHIVE="$SCRIPT_DIR/ui-dist.zip"
UNPACK_DIR="$SCRIPT_DIR/ui-dist"
APP_SUBDIR="thales-edge"   # folder inside the zip

# 1. Unpack once
if [ ! -d "$UNPACK_DIR/$APP_SUBDIR" ]; then
  echo "==> Unpacking Angular UI from $ARCHIVE ..."
  mkdir -p "$UNPACK_DIR"
  unzip -q "$ARCHIVE" -d "$UNPACK_DIR"
fi

APP_DIR="$UNPACK_DIR/$APP_SUBDIR"

# 2. Serve static files
echo "==> Serving Angular UI from: $APP_DIR"
echo "    Open http://localhost:4200 in your browser"

npx serve "$APP_DIR" -l 4200
```

Then:

```bash
cd /c/Users/TAP3507/source/shell-script-files
chmod +x start-angular.sh
./start-angular.sh
```

---

## 3️⃣ What you give to other people

Exactly two files:

1. `ui-dist.zip` (your “jar-like” Angular artifact)
    
2. `start-angular.sh` (the launcher)
    

They:

```bash
./start-angular.sh
```

Script unzips `ui-dist.zip` once and serves the app on `http://localhost:4200`.

If you’d like, next step I can line up **all** your artifacts:

- 3 Spring Boot fat jars + their start scripts
    
- Angular `ui-dist.zip` + `start-angular.sh`
    

so you have a clean, consistent story to share.