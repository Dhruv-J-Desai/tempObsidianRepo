- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Yeah, it’s not that Deephaven _isn’t_ returning a PSK – it’s that your script isn’t **finding** it in the log, so it’s stuck in the “Waiting for Deephaven PSK…” loop.

Let’s make `start-deephaven.sh`:

1. Definitely write logs to the right file
    
2. Search for the right pattern (`psk=`)
    
3. Time out with a helpful error instead of hanging forever
    
4. Extract the PSK robustly
    

Here’s a **drop-in replacement** for your `start-deephaven.sh` that does all of that.

---

## ✅ Updated `start-deephaven.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

##############################################
# 0. CONFIG - EDIT THESE PATHS
##############################################

DEEPhaven_VENV_DIR="C:/Users/TAP3507/source/projects/deephaven-pocs/dh-venv"
DEEPhaven_LOG="/c/tmp/deephaven.log"       # make sure /c/tmp exists, or change path

# Deephaven dirs
export DEEPHAVEN_DATA_DIR="c:/dhdata/data"
export DEEPHAVEN_CACHE_DIR="c:/dhdata/cache"
export DEEPHAVEN_CONFIG_DIR="c:/dhdata/config"
export DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200"
export DEEPHAVEN_AUTH_TYPE="psk"
export PY_STARTUP="/c/Users/TAP3507/.deephaven/startup.d"

##############################################
# 1. START DEEPHAVEN AND GRAB PSK
##############################################

echo "==> Activating Deephaven venv..."
source "$DEEPhaven_VENV_DIR/Scripts/activate"

echo "==> Starting Deephaven server on port 10000..."
# ensure log dir exists
mkdir -p "$(dirname "$DEEPhaven_LOG")"
: > "$DEEPhaven_LOG"   # truncate log file

deephaven server --port 10000 > "$DEEPhaven_LOG" 2>&1 &
DEEPhaven_PID=$!
echo "Deephaven PID: $DEEPhaven_PID"
echo "Log file     : $DEEPhaven_LOG"

echo "==> Waiting for Deephaven PSK (looking for 'psk=' in log)..."

SECONDS=0
while ! grep -q "psk=" "$DEEPhaven_LOG"; do
  sleep 1
  if (( SECONDS > 60 )); then
    echo "ERROR: Timed out after 60s waiting for PSK."
    echo "------ First 40 lines of $DEEPhaven_LOG ------"
    sed -n '1,40p' "$DEEPhaven_LOG"
    echo "----------------------------------------------"
    exit 1
  fi
done

# Extract PSK: take last occurrence of 'psk=...' and strip everything before '='
PSK=$(grep -o 'psk=[^&[:space:]]*' "$DEEPhaven_LOG" | tail -n 1 | cut -d= -f2)

if [[ -z "$PSK" ]]; then
  echo "ERROR: Found 'psk=' in log but could not extract token."
  echo "Last matching line:"
  grep "psk=" "$DEEPhaven_LOG" | tail -n 1
  exit 1
fi

echo "==> Deephaven PSK detected: $PSK"

# IMPORTANT: print ONLY the PSK as the final output so other scripts can capture it
echo "$PSK"
```

---

## 🔧 How this fixes your “stuck” state

- The loop now looks for **`psk=`** in the log:
    
    ```bash
    while ! grep -q "psk=" "$DEEPhaven_LOG"; do ...
    ```
    
- If after 60 seconds nothing with `psk=` shows up, it:
    
    - prints first 40 lines of the log
        
    - exits with an error instead of hanging forever
        
- When it finds a line like:
    
    ```text
    http://localhost:10000/?psk=16uqvhgz4p88d&something=...
    ```
    
    it will extract `16uqvhgz4p88d` correctly via:
    
    ```bash
    grep -o 'psk=[^&[:space:]]*' ... | cut -d= -f2
    ```
    
- The **last line printed is just the PSK**, so in `start-dh-orchestrator.sh` you can safely do:
    
    ```bash
    PSK=$(./start-deephaven.sh)
    export DH_PSK="$PSK"
    ```
    

---

## 🧪 Quick test

1. Run just:
    
    ```bash
    ./start-deephaven.sh
    ```
    
2. You should see:
    
    ```text
    ==> Activating Deephaven venv...
    ==> Starting Deephaven server on port 10000...
    ...
    ==> Deephaven PSK detected: 16uqvhgz4p88d
    16uqvhgz4p88d
    ```
    
    (The last line is the raw PSK)
    

If you instead see the 60-second timeout dump, then Deephaven isn’t logging the PSK in the file we think it is – but with the log dump printed you’ll at least be able to see _what_ it is printing and adjust the `grep` pattern accordingly.

But with the usual Deephaven output, this version will unblock you and let the orchestrator script pick up the PSK cleanly.