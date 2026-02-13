Yep — from the screenshots, your script is breaking for **multiple real reasons** (not just “job vs standard”). The main ones are in the **logic + bash syntax/escaping**.

Below are the exact break points I can see and how to fix them.

---

## 1) Your “wait for agent install” is not actually waiting

You wrote:

`while [ -z $datadoginstalled ]; do   if [ -e "/etc/datadog-agent/datadog.yaml" ]; then     datadoginstalled=TRUE   else     agent_error "The agent was not installed..."     exit $?   fi done`

**Problem:** On the _first_ check, if the file isn’t there yet (very common), you immediately call `agent_error` and exit. There is **no sleep/retry**, so this will fail on fresh clusters (job clusters especially).

✅ Fix it like this (real wait + timeout):

`datadoginstalled="" tries=0 max_tries=24  # 24 * 5s = 120s  while [ -z "$datadoginstalled" ] && [ $tries -lt $max_tries ]; do   if [ -f "/etc/datadog-agent/datadog.yaml" ]; then     datadoginstalled="TRUE"   else     tries=$((tries+1))     sleep 5   fi done  if [ -z "$datadoginstalled" ]; then   echo "Datadog agent not installed after timeout; skipping config to avoid cluster failure"   exit 0 fi`

(Notice `exit 0` — don’t kill the cluster just because DD isn’t ready.)

---

## 2) Your `agent_error()` function body is not valid bash (it will error)

In your screenshot, `agent_error()` contains:

- a `curl ...` line
    
- then **a raw JSON block starting with `{` on the next line**
    
- then `DD_AGENT_INSTALL`
    
- then `return 3`
    

But you **did not** pass the JSON to curl using `-d` or `--data`.

So bash interprets:

`{   "alert_type": "error",   ... }`

as a **bash command group**, and the lines like `"alert_type": "error"` become commands → that causes **command not found / non-zero**.

✅ Fix pattern:

`agent_error() {   msg="$1"   echo "* Datadog Agent issue: $msg"    # DO NOT fail cluster if event post fails   curl -sS -X POST "https://api.us3.datadoghq.com/api/v1/events" \     -H "Accept: application/json" \     -H "Content-Type: application/json" \     -H "DD-API-KEY: ${DD_API_KEY}" \     -d "$(cat <<JSON {   "alert_type": "error",   "title": "Datadog init script failure",   "text": "${msg}",   "tags": ["databricks_cluster_id:${DB_CLUSTER_ID}", "databricks_cluster_name:${DB_CLUSTER_NAME}"] } JSON )" || true }`

Key points:

- JSON is passed via `-d`
    
- `|| true` so curl failure doesn’t kill init script
    
- don’t `return 3` unless you truly want cluster termination
    

---

## 3) You’re escaping variables like `\${DB_IS_DRIVER}` (likely preventing expansion)

I see patterns like:

- `if [[ \${DB_IS_DRIVER} = "TRUE" ]]; then`
    
- `CNTFILE=/tmp/\$\.tmp` (looks like you tried to write `/tmp/$$.tmp`)
    

If those backslashes **actually exist on the cluster**, then bash will treat them literally and variables won’t expand as expected.

✅ Fix:

- Don’t escape variables in the script that actually runs on the cluster.
    
- If you’re generating a script via `cat <<EOF`, use a **single-quoted heredoc** so you don’t need backslashes:
    

`cat <<'EOF' > /tmp/start_datadog_common_driver_node_config.sh # variables like ${DB_IS_DRIVER} will be expanded when THIS script runs, not while writing it EOF`

Then inside, write `${DB_IS_DRIVER}` normally.

Also fix the temp file:

`CNTFILE="/tmp/$$.tmp"`

---

## 4) You’re doing `exit $?` right after calling `agent_error`

Example:

`agent_error "The agent was not installed..." exit $?`

`agent_error` returns whatever the last command returns (curl often fails) → so your script exits non-zero → **cluster dies**.

✅ Fix: If you want to stop config but not kill cluster:

`agent_error "..." exit 0`

or if you really want failure, do explicit:

`agent_error "..." exit 1`

---

## 5) You’re restarting datadog-agent and then checking `systemctl is-active`, but not protecting it

This block:

`systemctl restart datadog-agent systemctl is-active --quiet datadog-agent 2>&1 if [ $? -ne 0 ]; then   agent_error ... else   echo "Agent restarted" fi exit $?`

If `systemctl` isn’t available (some images / contexts) or service name differs, it fails → your `exit $?` ends the whole script.

✅ Safer version:

`systemctl restart datadog-agent || true systemctl is-active --quiet datadog-agent && echo "Datadog agent active" || echo "Datadog agent not active (continuing)" exit 0`

---

# Why it “works sometimes” on standard cluster

Because standard clusters might:

- start slower / agent finishes first by chance
    
- reuse VM state
    
- already have `/etc/datadog-agent/datadog.yaml` present from earlier runs
    

Job clusters are fresh → your “wait” logic immediately trips.

---

# The fastest path to unblock you

1. Fix the **wait loop** (add retry/sleep/timeout, don’t exit non-zero)
    
2. Fix `agent_error()` (valid curl JSON `-d`, and don’t let curl failure kill cluster)
    
3. Remove the `\${...}` escapes (or switch to `cat <<'EOF'`)
    

If you paste the _actual_ text of your `datadog_common_driver_node_config.sh` (not screenshots), I can rewrite it cleanly into a “safe init script” version that:

- works on job + standard
    
- never kills cluster for a transient DD issue
    
- still logs enough to debug why DD didn’t come up