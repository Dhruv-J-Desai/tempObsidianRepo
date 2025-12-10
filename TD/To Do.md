- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Nice, glad it’s working now 🎉  
Let’s turn what you have into a clean, “production-ish” `start-kafka-producer-consumer.sh`.

Below is a full script that:

- Fetches a fresh Databricks token via the CLI
    
- Exports all the env vars your Spring Boot app expects (Kafka + Databricks)
    
- Starts the **jar** from a path relative to the script
    
- Has basic error-handling and clear logging
    

You can drop this into your `shell-script-files` folder as `start-kafka-producer-consumer.sh`.

---

```bash
#!/usr/bin/env bash
#
# Start the Kafka Producer / Consumer demo with all required env vars.
# - Fetches Databricks token via `databricks auth token`
# - Exports Kafka + Databricks env vars
# - Runs the Spring Boot jar
#

set -euo pipefail

###############################################
# 0. Resolve paths
###############################################

# Directory where this script lives
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Path to the jar (adjust name if you bump version)
JAR_PATH="${SCRIPT_DIR}/jars/kafka-producer-consumer-0.0.1-SNAPSHOT.jar"

if [[ ! -f "$JAR_PATH" ]]; then
  echo "ERROR: Jar not found at: $JAR_PATH"
  echo "       Make sure you built it and copied it to shell-script-files/jars/"
  exit 1
fi

###############################################
# 1. Databricks configuration
###############################################

# Workspace + SQL warehouse
export DATABRICKS_HOST="adb-3218410855619456.16.azuredatabricks.net"
export DATABRICKS_HTTP_PATH="/sql/1.0/warehouses/2987cd418bca5dd5"

# Optional: if you use a specific profile name in ~/.databrickscfg
# export DATABRICKS_CONFIG_PROFILE="default"

###############################################
# 2. Fetch fresh Databricks token via CLI
###############################################

echo "== Databricks auth =="
if ! command -v databricks >/dev/null 2>&1; then
  echo "ERROR: 'databricks' CLI not found on PATH."
  echo "       Install Databricks CLI v0.228+ and run 'databricks auth login https://${DATABRICKS_HOST}' once."
  exit 1
fi

echo "-> Fetching Databricks auth token via CLI..."
echo "   (Make sure you have already run 'databricks auth login https://${DATABRICKS_HOST}' on this machine)"

# If you use a profile, add:  --profile "$DATABRICKS_CONFIG_PROFILE"
TOKEN_JSON="$(databricks auth token "https://${DATABRICKS_HOST}" 2>&1 || true)"

# Extract "access_token" from the JSON using sed (no jq dependency)
DATABRICKS_TOKEN_VALUE="$(
  echo "$TOKEN_JSON" \
    | tr -d '\r' \
    | sed -n 's/.*"access_token"[[:space:]]*:[[:space:]]*"\([^"]\+\)".*/\1/p'
)"

if [[ -z "${DATABRICKS_TOKEN_VALUE}" ]]; then
  echo "ERROR: Could not extract Databricks token from CLI output."
  echo "Raw output from 'databricks auth token':"
  echo "----------------------------------------"
  echo "$TOKEN_JSON"
  echo "----------------------------------------"
  exit 1
fi

export DATABRICKS_TOKEN="${DATABRICKS_TOKEN_VALUE}"
echo "-> Databricks token acquired."

###############################################
# 3. Kafka / Confluent + SDK env vars
###############################################

echo "== Kafka / SDK env vars =="

# Confluent Cloud / Kafka cluster
export bootstrap_servers="pkc-k13op.canadacentral.azure.confluent.cloud:9092"
export ccloud_client_id="TestScopeClient"
export ccloud_client_secret="2Federate"
export ccloud_identity_pool="pool-NRkI"
export ccloud_kafka_cluster_id="lkc-ygvwwp"

# Consumer configuration
export group_id="cc01_sb_its_esp_tap3507_customer"
export concurrency="2"
export poll_timeout_ms="1500"
export backoff_delay_ms="1000"
export backoff_max_retries="3"

# Topics
export kafka_customer_topic="cc01_sbq_its_esp_tap3507_customer"
export kafka_trade_topic="cc01_sb_its_esp_tap3507_trade"

# OAuth / SASL for Kafka
export oauthbearer_token_endpoint_url="https://fedsit.rastest.tdbank.ca/as/token.oauth2"
export sub_claim_name="client_id"

# SSL truststore for Kafka clients
# NOTE: double-backslashes for Java on Windows
export ssl_truststore_location="C:\\Users\\TAP3507\\certs\\cacerts-updated"
export ssl_truststore_password="changeit"
export ssl_truststore_type="JKS"

echo "-> bootstrap_servers     = ${bootstrap_servers}"
echo "-> kafka_customer_topic  = ${kafka_customer_topic}"
echo "-> kafka_trade_topic     = ${kafka_trade_topic}"
echo "-> DATABRICKS_HOST       = ${DATABRICKS_HOST}"
echo "-> DATABRICKS_HTTP_PATH  = ${DATABRICKS_HTTP_PATH}"
echo

###############################################
# 4. Start the Spring Boot app
###############################################

echo "== Starting Kafka Producer/Consumer =="
echo "Jar: $JAR_PATH"
echo

# If you want to ensure Java 17 is used:
# export JAVA_HOME="/c/Users/TAP3507/tools/java/jdk-17"
# export PATH="$JAVA_HOME/bin:$PATH"

java -jar "$JAR_PATH"
EXIT_CODE=$?

echo
echo "Kafka Producer/Consumer exited with code: $EXIT_CODE"
exit "$EXIT_CODE"
```

---

### How to use

1. Make it executable (in Git Bash or WSL):
    

```bash
chmod +x start-kafka-producer-consumer.sh
```

2. Run it from **PowerShell** or **Git Bash**:
    

```bash
./start-kafka-producer-consumer.sh
```

(If you’re in PowerShell, you can run: `bash start-kafka-producer-consumer.sh`.)

3. The script will:
    

- fetch a fresh Databricks token using your existing CLI login
    
- export all the env vars your Spring Boot app expects
    
- launch `kafka-producer-consumer-0.0.1-SNAPSHOT.jar`
    

If you change jar version, warehouse ID, topics, etc., just edit the config section at the top.

If you want similar scripts for your **DeepHaven**, **orchestrator**, or **BI showcase** services, you can use this as a template and we can clone/adjust them.