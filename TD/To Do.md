- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Got it – thanks for the screenshots, they tell the story pretty clearly.

### What’s happening right now

From `start-deephaven.sh` + the log:

- Deephaven **starts fine**:
    
    ```text
    Server started on port 10000
    Deephaven is running at http://localhost:10000/?psk=lm2t3c... with authentication type ...
    Press Control-T to exit
    ```
    
- Then, very soon after, you always see:
    
    ```text
    Aborted!
    ```
    
- And your browser gets `ERR_CONNECTION_REFUSED` → nothing is listening on `localhost:10000`.
    

So Deephaven is **starting and then exiting**. On Linux, the `nohup ... &` + `disown` trick usually keeps it alive after the script exits, but on Git Bash / Windows it’s clearly still getting killed when the parent shell ends. That’s why we keep seeing “Aborted!” even with `nohup`.

In other words:

> The more we try to be clever and start/stop Deephaven from the same script, the more Windows/Git-Bash fights us.

At this point, the most reliable thing is to split responsibilities:

---

## ✅ Stable, simple workflow

### 1. One terminal purely for Deephaven (keep it open)

Make `start-deephaven.sh` **only** start Deephaven and never exit until you Ctrl-C.

```bash
#!/usr/bin/env bash
set -euo pipefail

DEEPhaven_VENV_DIR="C:/Users/TAP3507/source/projects/deephaven-pocs/dh-venv"
DEEPhaven_LOG="/c/tmp/deephaven.log"

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
: > "$DEEPhaven_LOG"

# run in foreground, let this terminal own it
deephaven server --port 10000 | tee -a "$DEEPhaven_LOG"
```

Usage:

- Open **Git Bash terminal #1**
    
- Run:
    
    ```bash
    ./start-deephaven.sh
    ```
    
- Leave this window open. You’ll see:
    
    ```text
    Deephaven is running at http://localhost:10000/?psk=XYZ...
    Press Control-T to exit
    ```
    
- Copy that URL → browser → UI works ✅
    

You only stop Deephaven when you actually press Ctrl-C in this window.

---

### 2. Other scripts assume Deephaven is already running

Now **stop trying to start Deephaven from `start-all.sh` or from the orchestrator script.** Instead:

- Deephaven runs separately in its own terminal.
    
- Your other scripts **only need the PSK**, which they can read from the log.
    

Example helper script to grab the latest PSK:

```bash
#!/usr/bin/env bash
set -euo pipefail

DEEPhaven_LOG="/c/tmp/deephaven.log"

PSK=$(grep -o 'psk=[^&[:space:]]*' "$DEEPhaven_LOG" | tail -n 1 | cut -d= -f2)

if [[ -z "$PSK" ]]; then
  echo "ERROR: Could not find PSK in $DEEPhaven_LOG" >&2
  exit 1
fi

echo "$PSK"
```

Call it `get-dh-psk.sh`.

Then in `start-dh-orchestrator.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
JAR_DIR="$SCRIPT_DIR/jars"

PSK=$("$SCRIPT_DIR/get-dh-psk.sh")
echo "Using Deephaven PSK: $PSK"

export DH_PSK="$PSK"
export SPRING_PROFILES_ACTIVE="dev"

java -jar "$JAR_DIR/dh-orchestrator-0.0.1-SNAPSHOT.jar"
```

Same pattern for BIShowcase, etc.

---

## Why I’m recommending this

On Windows/Git-Bash:

- Background processes + `nohup` + `disown` work inconsistently.
    
- Deephaven is clearly sensitive to losing its controlling shell (you keep getting “Aborted!”).
    
- Keeping Deephaven in a dedicated terminal is **exactly what you were doing manually before**, and it was rock solid.
    

So:

- **One terminal** = “Deephaven terminal” (run once, leave open)
    
- **Other scripts** = start Java jars, Angular, etc., and just **read PSK from the log** instead of trying to own the Deephaven process.
    

That avoids all the weird process-lifetime interactions and gets you a reliable setup.

If you’d like, I can now give you a clean final `start-all.sh` that:

1. Assumes Deephaven is running.
    
2. Reads PSK via `get-dh-psk.sh`.
    
3. Starts:
    
    - dh-orchestrator (jar)
        
    - BIShowcase backend (jar)
        
    - data-product-gen (jar)
        
    - prints out the Deephaven URL and PSK for you at the end.