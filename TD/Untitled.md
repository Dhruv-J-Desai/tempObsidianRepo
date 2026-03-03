Yep — that explains the job failure. If the `.bundle/.../files` folder is empty, then **your bundle deploy didn’t actually sync `notebook.py` into the workspace**, so the job can’t find it at runtime.

This is usually because **bundles only upload (“sync”) files that are included by the bundle’s sync rules**, and your `notebook.py` isn’t being included.

## Fix it the clean way (recommended)

### 1) Put your script under a `src/` folder

From your local `td-bundle/` folder:

```bash
mkdir -p src
mv notebook.py src/notebook.py
```

### 2) Update `databricks.yml` to point to the workspace uploaded path

Change:

```yaml
python_file: ${workspace.file_path}/notebook.py
```

to:

```yaml
python_file: ${workspace.file_path}/src/notebook.py
```

### 3) Add an explicit sync include (so it definitely uploads)

Add this to `databricks.yml` (top-level):

```yaml
sync:
  include:
    - "src/**"
```

### 4) Redeploy

```bash
databricks bundle deploy -t dev --profile adb-3218410855619456
```

---

## Verify it actually uploaded (don’t rely only on the UI)

Run:

```bash
databricks workspace ls "/Workspace/Users/robin.bajaj@dev-td.com/.bundle/td-bundle/dev/files" --profile adb-3218410855619456
databricks workspace ls "/Workspace/Users/robin.bajaj@dev-td.com/.bundle/td-bundle/dev/files/src" --profile adb-3218410855619456
```

You should see `notebook.py` under `src/`.

---

### Why the UI shows “No files”

Databricks sometimes doesn’t render “workspace files” the same way in the Workspace browser, but the **CLI `workspace ls` is the source of truth**. If `workspace ls` shows empty, it truly didn’t sync.

---

If you paste your current `databricks.yml` (just the `sync:` part if you already have one, plus the task), I’ll tell you exactly what to change line-by-line.