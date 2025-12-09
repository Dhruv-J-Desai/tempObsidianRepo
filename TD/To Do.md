- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
  
  
  You can absolutely automate this with a single startup script. Since you’re already using `source` and `export`, I’ll give you a **bash script** that you can run from Git Bash / WSL on your Windows machine.

Got it – I can see in the screenshot you’re using the **Databricks CLI**:

1. `databricks auth login --host https://adb-...azuredatabricks.net` (only when not logged in)
    
2. `databricks auth token --host https://adb-...azuredatabricks.net` (prints a JSON blob with the token)
    

Let’s bake **step 2** into the script and assume you’ve already done `databricks auth login` once on this machine (it’s interactive so we can’t fully automate that part without a service principal).

Below is a **complete updated `start-all.sh`** that now:

- Starts **Deephaven** and extracts PSK from the log
    
- Updates **Angular’s** `environment.ts`
    
- Starts **dh-orchestrator**
    
- Starts **BIShowcase-backend**
    
- Starts **data-product-gen**
    
- **Fetches a Databricks auth token via `databricks auth token` and exports it**
    
- Starts **kafka_producer_consumer** with that token
    

You just need to fill in your real paths and secrets.

---

## ✅ Full `start-all.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

###############################################################################
#                          CONFIGURE YOUR PATHS HERE
###############################################################################

# Python virtual env for Deephaven
DEEPhaven_VENV_DIR="/c/Users/TAP3507/source/dh-venv"                    # <<< CHANGE if needed
DEEPhaven_LOG="/c/tmp/deephaven.log"                                    # <<< CHANGE if you like

# Angular UI
ANGULAR_APP_DIR="/c/Users/TAP3507/source/td-angular-ui"                 # <<< CHANGE
ANGULAR_ENV_FILE="$ANGULAR_APP_DIR/src/environments/environment.ts"

# Java Spring Boot apps
DH_ORCHESTRATOR_DIR="/c/Users/TAP3507/source/dh-orchestrator"          # <<< CHANGE
BISHOWCASE_BACKEND_DIR="/c/Users/TAP3507/source/BIShowcase-backend"    # <<< CHANGE
DATA_PRODUCT_GEN_DIR="/c/Users/TAP3507/source/data-product-gen"        # <<< CHANGE
KAFKA_PRODUCER_CONSUMER_DIR="/c/Users/TAP3507/source/kafka-producer-consumer"  # <<< CHANGE


###############################################################################
#                        Deephaven Environment Variables
###############################################################################

export DEEPHAVEN_DATA_DIR="c:/dhdata/data"
export DEEPHAVEN_CACHE_DIR="c:/dhdata/cache"
export DEEPHAVEN_CONFIG_DIR="c:/dhdata/config"
export DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200"
export DEEPHAVEN_AUTH_TYPE="psk"
export PY_STARTUP="/c/Users/TAP3507/.deephaven/startup.d"


###############################################################################
#                      BIShowcase / Databricks ENV VARS
###############################################################################

# These mirror what you had in IntelliJ for BIShowcase
export bootstrap_servers="pkc-13p0p.canadacentral.azure.confluent.cloud:9092"
export cloud_client_secret="..."                            # <<< CHANGE
export cloud_client_id="..."                                # <<< CHANGE if used

# Databricks host – used by:
#  - BIShowcase (through your YAML)
#  - databricks CLI (to mint tokens)
export databricksHost="https://adb-321840855691456.16.azuredatabricks.net"  # <<< CHANGE
export DATABRICKS_HOST="$databricksHost"                   # for CLI

export oauthbearer_token_endpoint_url="https://login.microsoftonline.com/.../oauth2/v2.0/token" # <<< CHANGE
export poolSSOClientId="..."
export poolSSOClientSecret="..."
export federated_tdp="..."

# Add any other env vars you have in IntelliJ as:
# export VAR_NAME="value"


###############################################################################
#                     1. START DEEPHAVEN & CAPTURE PSK
###############################################################################

echo "==> Activating Deephaven venv..."
source "$DEEPhaven_VENV_DIR/Scripts/activate"

echo "==> Starting Deephaven on port 10000..."
deephaven server --port 10000 > "$DEEPhaven_LOG" 2>&1 &
DEEPhaven_PID=$!

echo "Deephaven PID: $DEEPhaven_PID"
echo "Log file     : $DEEPhaven_LOG"

echo "==> Waiting for PSK (looking for 'psk=' in the log)..."

# Wait until the log contains a URL with ?psk=...
while ! grep -q "psk=" "$DEEPhaven_LOG"; do
  sleep 1
done

# Take the last line that has psk= and extract the token after psk=
PSK_LINE=$(grep "psk=" "$DEEPhaven_LOG" | tail -n 1)
PSK=$(echo "$PSK_LINE" | sed -E 's/.*psk=([^&[:space:]]+).*/\1/')

echo "==> PSK FOUND: $PSK"

# Export so Java apps can use it if they read DH_PSK
export DH_PSK="$PSK"


###############################################################################
#                   2. UPDATE ANGULAR environment.ts
###############################################################################

echo "==> Updating Angular environment.ts at $ANGULAR_ENV_FILE"

cp "$ANGULAR_ENV_FILE" "${ANGULAR_ENV_FILE}.bak"

# Replace DEEPHAVEN_PSK: '...'
sed -i -E "s/(DEEPHAVEN_PSK:\s*')[^']*(')/\1$PSK\2/" "$ANGULAR_ENV_FILE"


###############################################################################
#                     3. START ANGULAR DEV SERVER
###############################################################################

echo "==> Starting Angular dev server..."
(
  cd "$ANGULAR_APP_DIR"
  # npm install   # uncomment if you want automatic install
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
#                     6. START data-product-gen
###############################################################################

echo "==> Starting data-product-gen..."
(
  cd "$DATA_PRODUCT_GEN_DIR"
  mvn spring-boot:run -Dspring-boot.run.profiles=dev
) &
DATA_PRODUCT_GEN_PID=$!


###############################################################################
#            7. FETCH DATABRICKS TOKEN FOR kafka_producer_consumer
###############################################################################

echo "==> Fetching Databricks auth token via CLI..."
echo "    (make sure you have already run 'databricks auth login --host $DATABRICKS_HOST' once on this machine)"

# We ask for JSON and extract the "token" field using Python
TOKEN_JSON=$(databricks auth token --host "$DATABRICKS_HOST" --output json)
DATABRICKS_TOKEN_VALUE=$(echo "$TOKEN_JSON" | python - << 'PY'
import sys, json
data = json.load(sys.stdin)
print(data.get("token", ""))
PY
)

if [[ -z "$DATABRICKS_TOKEN_VALUE" ]]; then
  echo "WARNING: Could not extract Databricks token from CLI output:"
  echo "$TOKEN_JSON"
else
  echo "==> Databricks token acquired."
fi

# Export for kafka_producer_consumer (adjust the var name to whatever your app expects)
export DATABRICKS_TOKEN="$DATABRICKS_TOKEN_VALUE"
export DATABRICKS_ACCESS_TOKEN="$DATABRICKS_TOKEN_VALUE"   # export both just in case


###############################################################################
#                     8. START kafka_producer_consumer
###############################################################################

echo "==> Starting kafka_producer_consumer..."
(
  cd "$KAFKA_PRODUCER_CONSUMER_DIR"
  mvn spring-boot:run -Dspring-boot.run.profiles=dev
) &
KAFKA_PROD_CONS_PID=$!


###############################################################################
#                                SUMMARY
###############################################################################

echo ""
echo "==========================================================="
echo "  ALL SERVICES STARTED"
echo "-----------------------------------------------------------"
echo "  Deephaven PID             : $DEEPhaven_PID"
echo "  Angular PID               : $ANGULAR_PID"
echo "  dh-orchestrator PID       : $ORCH_PID"
echo "  BIShowcase-backend PID    : $BISHOWCASE_PID"
echo "  data-product-gen PID      : $DATA_PRODUCT_GEN_PID"
echo "  kafka_producer_consumer   : $KAFKA_PROD_CONS_PID"
echo "-----------------------------------------------------------"
echo "  Deephaven PSK             : $PSK"
echo "  Databricks host           : $DATABRICKS_HOST"
echo "==========================================================="
echo ""

# Keep script alive until any child exits
wait
```

---

## How this replaces your manual steps

### Before

1. Run `databricks auth login ...` (sometimes)
    
2. Run `databricks auth token --host ...`
    
3. Copy the long JWT token
    
4. Paste into IntelliJ env var for `kafka_producer_consumer`
    
5. Start the app from IDE
    

### Now

- You just run:
    

```bash
chmod +x start-all.sh       # once
./start-all.sh
```

The script:

- Calls `databricks auth token --host ... --output json`
    
- Extracts `.token`
    
- Exports it as `DATABRICKS_TOKEN` and `DATABRICKS_ACCESS_TOKEN`
    
- Starts the `kafka_producer_consumer` app with that token in its environment
    

> First time on a new machine, you still need to run once manually:
> 
> ```bash
> databricks auth login --host https://adb-3218...azuredatabricks.net
> ```
> 
> so the CLI has a profile to use. After that, the script just works.

---

If you tell me the **exact env var name** your `kafka_producer_consumer` app reads (from the IntelliJ Run/Debug config, e.g. `DATABRICKS_HTTP_PATH`, `DATABRICKS_TOKEN`, etc.), I can tweak that export line to match it perfectly.