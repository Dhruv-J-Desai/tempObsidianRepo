- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Right, and that explains the confusion:  
`start-deephaven.sh` **does** start Deephaven, but the way we’re _using_ it is what’s killing the server, which is why the UI can’t connect.

Let me untangle what’s happening and then give you a clean, working pair of scripts.

---

## 🔍 What’s going wrong

From your log:

```text
Server started on port 10000
Deephaven is running at http://localhost:10000/?psk=8lz0e61c56ne …
Press Control-T to exit
Aborted!
```

So:

- Deephaven starts ✅
    
- Immediately prints a valid PSK ✅
    
- Then prints **“Aborted!”** and exits ❌
    

That “Aborted!” happens when the Deephaven process gets a signal (like when its parent shell dies).

The key detail: earlier in `start-dh-orchestrator.sh` you had:

```bash
PSK=$(./start-deephaven.sh)
```

That runs `start-deephaven.sh` in a **subshell**. Inside that subshell:

- Deephaven is started in the background (`&`)
    
- We wait for PSK in the log, echo it
    
- The subshell exits
    

When that subshell exits, **all its background jobs get killed** → Deephaven prints “Aborted!” and disappears.  
By the time your browser tries `http://localhost:10000/?psk=...`, nothing is listening → `ERR_CONNECTION_REFUSED`.

Even if you now run `start-deephaven.sh` directly, you probably did a mix of both, so the log is showing old “Aborted!” runs too.

---

## ✅ Clean solution: no subshell, PSK via file

We’ll do this:

1. `start-deephaven.sh`
    
    - Starts Deephaven (still in background)
        
    - Waits for `psk=` in the log
        
    - Writes PSK to a temp file (`/c/tmp/dh_psk`)
        
    - **Does NOT** run in a subshell; we call it directly.
        
2. `start-dh-orchestrator.sh`
    
    - Calls `./start-deephaven.sh` (normal call, **no** `$()`)
        
    - Reads PSK from that temp file
        
    - Exports `DH_PSK` and starts the orchestrator jar
        

### 1️⃣ `start-deephaven.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Root directories of your projects
DEEPhaven_VENV_DIR="C:/Users/TAP3507/source/projects/deephaven-pocs/dh-venv"
DEEPhaven_LOG="/c/tmp/deephaven.log"
PSK_FILE="/c/tmp/dh_psk"

# Deephaven dirs
export DEEPHAVEN_DATA_DIR="c:/dhdata/data"
export DEEPHAVEN_CACHE_DIR="c:/dhdata/cache"
export DEEPHAVEN_CONFIG_DIR="c:/dhdata/config"
export DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200"
export DEEPHAVEN_AUTH_TYPE="psk"
export PY_STARTUP="/c/Users/TAP3507/.deephaven/startup.d"

echo "==> Activating Deephaven venv..."
source "$DEEPhaven_VENV_DIR/Scripts/activate"

echo "==> Starting Deephaven server on port 10000..."
mkdir -p "$(dirname "$DEEPhaven_LOG")"
: > "$DEEPhaven_LOG"   # truncate log
rm -f "$PSK_FILE"

# Start in background
deephaven server --port 10000 > "$DEEPhaven_LOG" 2>&1 &
DEEPhaven_PID=$!
echo "Deephaven PID: $DEEPhaven_PID"
echo "Log file     : $DEEPhaven_LOG"

echo "==> Waiting for Deephaven PSK..."
SECONDS=0
while ! grep -q "psk=" "$DEEPhaven_LOG"; do
  sleep 1
  if (( SECONDS > 60 )); then
    echo "ERROR: Timed out after 60s waiting for PSK."
    exit 1
  fi
done

PSK=$(grep -o 'psk=[^&[:space:]]*' "$DEEPhaven_LOG" | tail -n 1 | cut -d= -f2)

if [[ -z "$PSK" ]]; then
  echo "ERROR: could not extract PSK."
  exit 1
fi

echo "==> Deephaven PSK detected: $PSK"

# Save PSK for other scripts
echo "$PSK" > "$PSK_FILE"

# Do NOT exit the shell as a subshell; we just finish and leave Deephaven running in background
```

Running this directly:

```bash
./start-deephaven.sh
```

- Starts Deephaven and leaves it running
    
- Writes PSK into `/c/tmp/dh_psk`
    

You should now be able to manually:

```bash
cat /c/tmp/dh_psk
```

copy that PSK and open `http://localhost:10000/?psk=<value>` in the browser → UI should load.

---

### 2️⃣ `start-dh-orchestrator.sh`

Now adjust the orchestrator script:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
JAR_DIR="$SCRIPT_DIR/jars"
PSK_FILE="/c/tmp/dh_psk"

echo "==> Starting Deephaven..."
./start-deephaven.sh

PSK=$(cat "$PSK_FILE")
echo "==> Using Deephaven PSK: $PSK"

export DH_PSK="$PSK"
export SPRING_PROFILES_ACTIVE="dev"

JAR_PATH="$JAR_DIR/dh-orchestrator-0.0.1-SNAPSHOT.jar"
echo "==> Starting dh-orchestrator..."
java -jar "$JAR_PATH"
```

Notice: **no** `PSK=$(./start-deephaven.sh)` anymore.  
We call `start-deephaven.sh` normally, so the Deephaven process lives in the same shell and isn’t killed when that tiny subshell exits.

---

## TL;DR

- Your UI fails because Deephaven is starting and then **exiting with “Aborted!”**.
    
- That’s caused by running `start-deephaven.sh` inside command substitution (`PSK=$(...)`), which kills its background jobs when the subshell exits.
    
- Fix: call `start-deephaven.sh` directly and share the PSK via a temp file or some other IPC, not via subshell capture.
    

Once you wire it up like above, you should see:

1. `start-deephaven.sh` runs, leaves `deephaven` alive
    
2. `/c/tmp/dh_psk` contains a PSK
    
3. Browser can open `http://localhost:10000/?psk=<that>`
    
4. Orchestrator/Angular can also use that PSK via `DH_PSK`.