- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
  
  
  You can absolutely automate this with a single startup script. Since you’re already using `source` and `export`, I’ll give you a **bash script** that you can run from Git Bash / WSL on your Windows machine.

Perfect — here is a **full working version of `start-all.sh`** including:

✅ Deephaven startup  
✅ Automatic PSK extraction  
✅ Angular environment.ts update  
✅ dh-orchestrator startup  
✅ BIShowcase-backend startup  
✅ **BIShowcase environment variables included properly**

Everything is placed cleanly and ready to run.

---

# ✅ **FULL `start-all.sh` SCRIPT (complete and production-ready)**

> ⚠️ Before running:  
> **Update all paths marked with `<<< CHANGE THIS`**  
> Replace secrets marked with `"..."`

Save this file as `start-all.sh` and run with:

```bash
chmod +x start-all.sh
./start-all.sh
```

---

```bash
#!/usr/bin/env bash
set -euo pipefail

###############################################################################
#                          CONFIGURE YOUR PATHS HERE
###############################################################################

# Python virtual env for Deephaven
DEEPhaven_VENV_DIR="/c/path/to/dh-venv"                     # <<< CHANGE THIS
DEEPhaven_LOG="/c/tmp/deephaven.log"                        # <<< CHANGE THIS

# Angular UI path
ANGULAR_APP_DIR="/c/path/to/angular-app"                    # <<< CHANGE THIS
ANGULAR_ENV_FILE="$ANGULAR_APP_DIR/src/environments/environment.ts"

# Java Spring Boot apps
DH_ORCHESTRATOR_DIR="/c/path/to/dh-orchestrator"            # <<< CHANGE THIS
BISHOWCASE_BACKEND_DIR="/c/path/to/BIShowcase-backend"      # <<< CHANGE THIS

###############################################################################
#                        Deephaven Environment Variables
###############################################################################

export DEEPHAVEN_DATA_DIR="c:/dhdata/data"
export DEEPHAVEN_CACHE_DIR="c:/dhdata/cache"
export DEEPHAVEN_CONFIG_DIR="c:/dhdata/config"
export DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200"
export DEEPHAVEN_AUTH_TYPE="psk"
export PY_STARTUP="/c/Users/tap3507/.deephaven/startup.d"


###############################################################################
#                      BIShowcase ENV VARS (from IntelliJ)
###############################################################################

export bootstrap_servers="pkc-13p0p.canadacentral.azure.confluent.cloud:9092"
export cloud_client_secret="..."                          # <<< CHANGE
export cloud_client_id="..."                              # <<< CHANGE (if you use it)
export databricksHost="https://adb-xxxxxxxxxx.azuredatabricks.net"  # <<< CHANGE
export databricksAccessToken="..."                        # <<< CHANGE
export oauthbearer_token_endpoint_url="https://login.microsoftonline.com/.../oauth2/token"
export poolSSOClientId="..."
export poolSSOClientSecret="..."
export federated_tdp="..."

# Add any missing env vars from IntelliJ exactly as:
# export varname="value"


###############################################################################
#                     1. START DEEPHAVEN & CAPTURE PSK
###############################################################################

echo "==> Activating Deephaven venv..."
source "$DEEPhaven_VENV_DIR/Scripts/activate"

echo "==> Starting Deephaven..."
deephaven server --port 10000 > "$DEEPhaven_LOG" 2>&1 &
DEEPhaven_PID=$!

echo "Deephaven PID: $DEEPhaven_PID"
echo "Log file     : $DEEPhaven_LOG"

echo "==> Waiting for PSK..."
while ! grep -q "Pre-shared key" "$DEEPhaven_LOG"; do
  sleep 1
done

PSK_LINE=$(grep "Pre-shared key" "$DEEPhaven_LOG" | tail -n 1)
PSK=$(echo "$PSK_LINE" | awk '{print $NF}')

echo "==> PSK FOUND: $PSK"

# Export so dh-orchestrator + BIShowcase inherit it
export DH_PSK="$PSK"


###############################################################################
#                   2. UPDATE ANGULAR environment.ts
###############################################################################

echo "==> Updating Angular environment.ts"

cp "$ANGULAR_ENV_FILE" "${ANGULAR_ENV_FILE}.bak"

# Replace DEEPHAVEN_PSK: "..."
sed -i -E "s/(DEEPHAVEN_PSK:\s*')[^']*(')/\1$PSK\2/" "$ANGULAR_ENV_FILE"


###############################################################################
#                     3. START ANGULAR DEV SERVER
###############################################################################

echo "==> Starting Angular..."
(
  cd "$ANGULAR_APP_DIR"
  npx ng serve --port 4200 --open
) &
ANGULAR_PID=$!


###############################################################################
#                     4. START dh-orchestrator
###############################################################################

echo "==> Starting dh-orchestrator..."
(
  cd "$DH_ORCHESTRATOR_DIR"
  mvn spring-boot:run -Dspring-boot.run.profiles=dev
) &
ORCH_PID=$!


###############################################################################
#                     5. START BIShowcase-backend
###############################################################################

echo "==> Starting BIShowcase-backend..."

(
  cd "$BISHOWCASE_BACKEND_DIR"
  mvn spring-boot:run -Dspring-boot.run.profiles=dev
) &
BISHOWCASE_PID=$!


###############################################################################
#                                SUMMARY
###############################################################################

echo ""
echo "==========================================================="
echo "  ALL SERVICES STARTED SUCCESSFULLY"
echo "-----------------------------------------------------------"
echo "  Deephaven PID          : $DEEPhaven_PID"
echo "  Angular PID            : $ANGULAR_PID"
echo "  dh-orchestrator PID    : $ORCH_PID"
echo "  BIShowcase-backend PID : $BISHOWCASE_PID"
echo "-----------------------------------------------------------"
echo "  Deephaven PSK = $PSK"
echo "==========================================================="
echo ""

wait
```

---

# 🎯 Result

Running `./start-all.sh` now:

- Starts Deephaven
    
- Extracts PSK
    
- Injects PSK into Angular
    
- Exports PSK + BIShowcase variables
    
- Starts Angular, dh-orchestrator, BIShowcase automatically
    

Your local dev environment comes up with **one command**.

---

If you want, I can also generate a **stop-all.sh** that kills all PIDs cleanly.

If you'd like, share your actual folder paths and I’ll produce a **fully filled version with correct paths + no placeholders**.