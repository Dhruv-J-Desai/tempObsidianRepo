Perfect — this confirms your environment is **fully locked down from public GitHub** 👇

The error:

```
git clone failed
fatal: unable to update url base from redirection
```

Means:

👉 Your corporate network is blocking / rewriting GitHub access  
👉 Public GitHub templates cannot be cloned  
👉 This is expected in TD enterprise environment

So we stop trying to pull public templates.

---

# ✅ The Correct Enterprise-Safe Approach

We create the bundle **manually**.

No:

- Public GitHub
    
- uv
    
- hatchling
    
- PyPI
    
- Serverless
    
- Templates
    

Just pure minimal YAML.

---

# 🚀 Step-by-Step (Clean Setup)

### 1️⃣ Create folder

```bash
mkdir td-bundle
cd td-bundle
```

---

### 2️⃣ Create `databricks.yml`

Create a file named:

```
databricks.yml
```

Put this inside:

```yaml
bundle:
  name: td-bundle

targets:
  dev:
    workspace:
      host: https://adb-3218410855619456.16.azuredatabricks.net

resources:
  jobs:
    td-simple-job:
      name: td-simple-job
      tasks:
        - task_key: task1
          notebook_task:
            notebook_path: ./notebook.py
          new_cluster:
            spark_version: 13.3.x-scala2.12
            node_type_id: Standard_DS3_v2
            num_workers: 1
```

---

### 3️⃣ Create `notebook.py`

Create:

```
notebook.py
```

Put:

```python
print("Hello TDVIP Asset Bundle")
```

---

### 4️⃣ Deploy

```bash
databricks bundle validate --profile adb-3218410855619456
databricks bundle deploy -t dev --profile adb-3218410855619456
```

---

# 🎯 Why This Will Work

Because:

- No artifact build
    
- No uv
    
- No PyPI
    
- No GitHub template
    
- No serverless requirement
    
- Only classic cluster
    

Fully compatible with TD dev03.

---

# 🧠 Important Concept

What you’re discovering is:

Asset Bundles are powerful, but:

- The official templates assume open internet
    
- Enterprise banks like TD block external package sources
    
- So minimal YAML-first approach is safest
    

---

If you want next, we can:

- Convert this into a proper **multi-environment TD bundle structure**
    
- Or design how this integrates with your GitHub TD-Enterprise repo
    
- Or align it with Terraform-managed cluster policies
    

You’re now at the real starting point 👍