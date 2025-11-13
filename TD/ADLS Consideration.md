Ahh yes — I remember what you’re referring to.

You previously used a command that **did NOT create a PAT**, but instead generated a **temporary Azure AD access token** for Databricks.  
Those AAD tokens **always last ~1 hour (sometimes up to 2 hours)**.

Here is the exact command you used before:

---

# ✅ **The Command You Previously Used (AAD Token for Databricks)**

```bash
az account get-access-token --resource 2ff814a6-3304-4ab8-85cb-cd0e6f879c1d --query accessToken -o tsv
```

This generates an **Azure AD OAuth token** for the Databricks resource:

- Resource ID: `2ff814a6-3304-4ab8-85cb-cd0e6f879c1d`
    
- Lifetime: **~1 hour**
    
- Purpose: Used as _Bearer token_ in REST API
    
- NOT a Databricks PAT  
    (PATs last days/months — OAuth tokens do not)
    

---

# 🎯 **How You Used It Earlier to Call Databricks**

You likely did something like:

```bash
TOKEN=$(az account get-access-token --resource 2ff814a6-3304-4ab8-85cb-cd0e6f879c1d --query accessToken -o tsv)

curl -H "Authorization: Bearer $TOKEN" \
     https://adb-<workspace>.azuredatabricks.net/api/2.0/clusters/list
```

This works because Databricks accepts AAD tokens.

---

# ❗ Important

This method **cannot generate a PAT**.  
AAD tokens cannot be converted to PAT tokens.

They only authenticate API calls directly.

---

# ⭐ If You Want: I Can Give You a Full Script

Here is a Git Bash script that:

1. logs into Azure
    
2. retrieves a fresh Databricks AAD token
    
3. prints it
    
4. uses it for any API call
    

---

# 🟢 Ready-To-Use Script (Token Lasts ~1–2 Hours)

```bash
#!/usr/bin/env bash

WORKSPACE_URL="https://adb-XXXXXXXXX.azuredatabricks.net"

echo "Logging in to Azure..."
az login --use-device-code

echo "Fetching temporary Databricks AAD access token..."
TOKEN=$(az account get-access-token \
  --resource 2ff814a6-3304-4ab8-85cb-cd0e6f879c1d \
  --query accessToken -o tsv)

echo "Token acquired:"
echo "$TOKEN"

echo "Testing token with Databricks API..."
curl -s -H "Authorization: Bearer $TOKEN" \
     "$WORKSPACE_URL/api/2.0/clusters/list" | jq
```

---

# ✔️ Summary

|Method|Type|Duration|You Used It Before?|
|---|---|---|---|
|`az account get-access-token`|**AAD OAuth token**|~1–2 hrs|✔️ Yes|
|Databricks PAT|Long-lived (days–months)|Requires PAT creation|✖️ No|

---

If you want, I can also show you:

✅ how to auto-refresh the token every hour  
✅ how to generate a long-term PAT _using the AAD token_  
(works only if your workspace admin allows PAT creation)

Just tell me what you need next.