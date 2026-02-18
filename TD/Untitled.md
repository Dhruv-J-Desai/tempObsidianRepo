Yep — that happens because **Datadog’s “level” facet is not being parsed from your custom log line**, so Datadog falls back to **status=info** for everything (even if the text contains `WARNING` / `ERROR`).

So your formatter is fine for splitting events, but **Datadog isn’t extracting severity**.

You said **Option A** (you can’t change the Datadog config), so the fix has to be **in the log line itself**.

---

## ✅ Fix (no Datadog config change): emit “status=” (and optionally “level=”) in the message

Datadog commonly auto-detects severity when it sees things like:

- `status:error`
    
- `status=error`
    
- `[ERROR]`
    
- `ERROR:` at the start (sometimes)
    

Best reliable approach: **include `status=<lowercase>`**.

### Update your formatter to include `status=...`

```python
class DDStatusFormatter(logging.Formatter):
    def formatTime(self, record, datefmt=None):
        dt = datetime.utcfromtimestamp(record.created)
        return dt.strftime("%Y-%m-%d %H:%M:%S.%f")[:-3]

    def format(self, record):
        # Map python levels -> datadog-ish statuses
        status = {
            "DEBUG": "debug",
            "INFO": "info",
            "WARNING": "warn",
            "ERROR": "error",
            "CRITICAL": "critical",
        }.get(record.levelname, "info")

        record.dd_status = status
        return super().format(record)

formatter = DDStatusFormatter(
    "%(asctime)s | status=%(dd_status)s | level=%(levelname)s | %(name)s | %(message)s"
)
```

Now Datadog has an easy token to pick up (`status=error`, `status=warn`, etc.).

---

## Why Datadog showed everything as INFO

Because your logs are being ingested as **plain text**, and unless Datadog is told how to parse your “level” field (via pipeline/grok/remap), it won’t automatically treat `WARNING` / `ERROR` in the middle of the line as severity.

So it keeps defaulting to “info”.

---

## Quick test line examples Datadog usually classifies correctly

Your logs will look like:

```
2026-02-18 17:24:04.567 | status=error | level=ERROR | TDVIP_APP | Testing ERROR level
```

That _usually_ gets categorized correctly without config changes.

---

If after this it’s **still all INFO**, it means your org’s Datadog pipeline is overriding status (some pipelines force “info” unless JSON attribute exists). In that case, tell me: are these logs showing up under `source:spark` like earlier?