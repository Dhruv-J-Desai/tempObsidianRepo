- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
  
  
  You can absolutely automate this with a single startup script. Since you’re already using `source` and `export`, I’ll give you a **bash script** that you can run from Git Bash / WSL on your Windows machine.

### What this script will do

1. Activate your Python venv and start Deephaven on port 10000.
    
2. Wait until Deephaven prints the PSK, then:
    
    - Extract the PSK from the log.
        
    - Inject it into your Angular `environment.ts`.
        
    - Export it as `DH_PSK` for both Spring Boot apps.
        
3. Start:
    
    - The Angular dev server.
        
    - `dh-orchestrator` Spring Boot app.
        
    - `BIShowcase-backend` Spring Boot app.
        

All in one go.

---

## `start-all.sh` (bash script)

Adjust the paths marked with `<<< CHANGE THIS` for your machine, save this as `start-all.sh`, and make it executable.

```bash
#!/usr/bin/env bash
set -euo pipefail

##############################################
# 0. CONFIG – EDIT THESE PATHS
##############################################

# Root directories of your projects
DEEPhaven_VENV_DIR="/c/path/to/dh-venv"                            # <<< CHANGE THIS
DEEPhaven_LOG="/c/tmp/deephaven.log"                               # <<< CHANGE THIS (any writable location)

ANGULAR_APP_DIR="/c/path/to/angular-app"                           # <<< CHANGE THIS
ANGULAR_ENV_FILE="$ANGULAR_APP_DIR/src/environments/environment.ts" # <<< CHANGE THIS

DH_ORCHESTRATOR_DIR="/c/path/to/dh-orchestrator"                   # <<< CHANGE THIS
BISHOWCASE_BACKEND_DIR="/c/path/to/BIShowcase-backend"             # <<< CHANGE THIS

# Deephaven dirs
export DEEPHAVEN_DATA_DIR="c:/dhdata/data"
export DEEPHAVEN_CACHE_DIR="c:/dhdata/cache"
export DEEPHAVEN_CONFIG_DIR="c:/dhdata/config"
export DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200"
export DEEPHAVEN_AUTH_TYPE="psk"
export PY_STARTUP="/c/Users/tap3507/.deephaven/startup.d"


##############################################
# 1. START DEEPHAVEN AND GRAB PSK
##############################################

echo "==> Activating Deephaven venv..."
# Typical Windows venv structure under Git Bash:
source "$DEEPhaven_VENV_DIR/Scripts/activate"

echo "==> Starting Deephaven server on port 10000..."
# Start Deephaven in background, log to file
deephaven server --port 10000 > "$DEEPhaven_LOG" 2>&1 &
DEEPhaven_PID=$!
echo "Deephaven PID: $DEEPhaven_PID"
echo "Log file     : $DEEPhaven_LOG"

echo "==> Waiting for Deephaven PSK..."
# Adjust the grep pattern to match the exact log line Deephaven prints for PSK
# Commonly it prints something like: "Pre-shared key: abc123..."
while ! grep -q "Pre-shared key" "$DEEPhaven_LOG"; do
  sleep 1
done

# Extract last occurrence of the PSK line and take the last "word" as token
PSK_LINE=$(grep "Pre-shared key" "$DEEPhaven_LOG" | tail -n 1)
PSK=$(echo "$PSK_LINE" | awk '{print $NF}')

echo "==> Deephaven PSK detected: $PSK"


##############################################
# 2. UPDATE ANGULAR environment.ts
##############################################

if [[ ! -f "$ANGULAR_ENV_FILE" ]]; then
  echo "ERROR: Angular environment.ts not found at:"
  echo "  $ANGULAR_ENV_FILE"
  exit 1
fi

echo "==> Backing up Angular environment.ts..."
cp "$ANGULAR_ENV_FILE" "${ANGULAR_ENV_FILE}.bak"

echo "==> Injecting PSK into Angular environment.ts..."

# We assume you have a line like:
#   DEEPHAVEN_PSK: "...",
# We replace the content between the quotes with the new PSK.
# Note: On Git Bash sed -i works like Linux; on plain Windows it differs.
sed -i -E "s/(DEEPHAVEN_PSK:\s*')[^']*(')/\1$PSK\2/" "$ANGULAR_ENV_FILE"

echo "Angular environment.ts updated."


##############################################
# 3. EXPORT DH_PSK FOR SPRING BOOT APPS
##############################################

export DH_PSK="$PSK"
echo "==> Exported DH_PSK for Spring Boot apps."


##############################################
# 4. START ANGULAR APP
##############################################

echo "==> Starting Angular dev server..."
(
  cd "$ANGULAR_APP_DIR"
  # Optional: install dependencies once; comment out if already done
  # npm install
  npx ng serve --port 4200 --open
) &

ANGULAR_PID=$!
echo "Angular PID: $ANGULAR_PID"


##############################################
# 5. START dh-orchestrator
##############################################

echo "==> Starting dh-orchestrator (Spring Boot)..."
(
  cd "$DH_ORCHESTRATOR_DIR"
  mvn spring-boot:run -Dspring-boot.run.profiles=dev
) &

ORCH_PID=$!
echo "dh-orchestrator PID: $ORCH_PID"


##############################################
# 6. START BIShowcase-backend
##############################################

echo "==> Starting BIShowcase-backend (Spring Boot)..."
(
  cd "$BISHOWCASE_BACKEND_DIR"
  mvn spring-boot:run -Dspring-boot.run.profiles=dev
) &

BISHOWCASE_PID=$!
echo "BIShowcase-backend PID: $BISHOWCASE_PID"


##############################################
# 7. DONE – KEEP SHELL OPEN
##############################################

echo
echo "===================================="
echo " All services started:"
echo "   Deephaven PID         = $DEEPhaven_PID"
echo "   Angular PID           = $ANGULAR_PID"
echo "   dh-orchestrator PID   = $ORCH_PID"
echo "   BIShowcase-backend PID= $BISHOWCASE_PID"
echo " Deephaven PSK          = $PSK"
echo "===================================="
echo "Use 'kill <PID>' to stop any service."
echo

# Wait for everything (so the script doesn't exit immediately)
wait
```

---

## How to use

1. Save this file as `start-all.sh` somewhere (e.g. in a “dev-tools” folder).
    
2. Make it executable (in Git Bash):
    
    ```bash
    chmod +x start-all.sh
    ```
    
3. Open **Git Bash** (or WSL), go to the folder containing the script:
    
    ```bash
    cd /c/path/to/dev-tools
    ./start-all.sh
    ```
    

That’s it – you’ll get:

- Deephaven running with PSK.
    
- Angular running with the PSK injected into `environment.ts`.
    
- Both Spring Boot apps running with `DH_PSK` in their environment.
    

If you want a PowerShell version as well (so you can double-click a `.ps1`), I can translate this script into PowerShell next.