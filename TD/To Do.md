- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
  
  
  You can absolutely automate this with a single startup script. Since you’re already using `source` and `export`, I’ll give you a **bash script** that you can run from Git Bash / WSL on your Windows machine.

Yeah, I see what’s happening now.

- The **CLI is definitely returning correct JSON** (your bottom screenshot proves it).
    
- But the little Python block in the script is still failing to parse it for some reason (probably because of some hidden characters / encoding / CRLF weirdness in the captured output).
    
- So `DATABRICKS_TOKEN_VALUE` ends up empty and you hit the `ERROR: Could not extract Databricks token` branch.
    

Let’s stop fighting Python here and just **grab the token with plain shell / sed**, which is much more forgiving.

---

## 🔁 Replace your whole token block with this

In `start-all.sh`, find the section that starts with:

```bash
#############################
# FETCH DATABRICKS TOKEN FOR kafka_producer_consumer
#############################
```

and replace everything down to the `if [[ -z "$DATABRICKS_TOKEN_VALUE" ]]` check with the following:

```bash
###############################################################################
#  FETCH DATABRICKS TOKEN FOR kafka_producer_consumer
###############################################################################

DATABRICKS_HOST="https://adb-3218410855619456.16.azuredatabricks.net"   # same as manual

echo "==> Fetching Databricks auth token via CLI..."
echo "    (make sure you have already run 'databricks auth login $DATABRICKS_HOST' once on this machine)"

# Use EXACTLY the same command you run manually (no --host flag)
TOKEN_JSON=$(databricks auth token "$DATABRICKS_HOST" 2>&1 || true)

# Extract the access_token value using sed (no Python, very robust)
# 1. Remove CR characters (from Windows newlines)
# 2. Grab the value after "access_token": "...."
DATABRICKS_TOKEN_VALUE=$(
  echo "$TOKEN_JSON" \
  | tr -d '\r' \
  | sed -n 's/.*"access_token"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p'
)

if [[ -z "$DATABRICKS_TOKEN_VALUE" ]]; then
  echo "ERROR: Could not extract Databricks token from CLI output:"
  echo "------ raw output from 'databricks auth token' ------"
  echo "$TOKEN_JSON"
  echo "-----------------------------------------------------"
  exit 1
else
  echo "==> Databricks token acquired."
fi

# Export for your apps (use the names you expect in application.yml / env)
export DATABRICKS_ACCESS_TOKEN="$DATABRICKS_TOKEN_VALUE"
export databricks_access_token="$DATABRICKS_TOKEN_VALUE"
```

Key points:

- It calls **exactly** `databricks auth token https://adb-…` like you do manually.
    
- It doesn’t care if there are extra spaces / newlines / minor formatting differences.
    
- `sed` just looks for `"access_token": "..."` and pulls out the `...` part.
    
- No Python, no JSON parsing issues.
    

---

After you make this change:

```bash
./start-all.sh
```

You should see:

```text
==> Fetching Databricks auth token via CLI...
==> Databricks token acquired.
```

…and then the rest of your apps start as usual, with `DATABRICKS_ACCESS_TOKEN` set to the same token you used to copy manually before.

If you tell me the exact env var name your Spring app reads (e.g. `DATABRICKS_TOKEN`, `databricks_access_token`), I can tailor the `export` lines to match it exactly, but the above will already work for most setups.