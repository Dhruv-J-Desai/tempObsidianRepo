# Databricks-safe “Job vs All-Purpose” detection + logger setup (no init-script changes needed)
# - Uses Databricks context tags (more reliable than DB_IS_JOB_CLUSTER)
# - Writes to /databricks/driver/logs for job compute, /app_logs for all-purpose by default
# - Prints a small diagnostics block so you can paste into Teams

import os
import sys
import json
import logging
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Any, Dict, Optional


# -------------------------
# 1) Context detection
# -------------------------
def _safe_get_dbutils_context_tags() -> Dict[str, str]:
    """
    Returns context tags if running in Databricks.
    Works in notebooks. If not available, returns {}.
    """
    try:
        ctx = dbutils.notebook.entry_point.getDbutils().notebook().getContext()  # type: ignore[name-defined]
        tags = ctx.tags()
        out = {}
        for k in tags.keySet():
            v = tags.get(k)
            if v.isDefined():
                out[str(k)] = v.get()
        return out
    except Exception:
        return {}


def _truthy_env(name: str) -> Optional[bool]:
    """Parse env var as True/False, return None if not set."""
    v = os.environ.get(name)
    if v is None or v == "":
        return None
    return v.strip().upper() in ("1", "TRUE", "YES", "Y", "T")


@dataclass
class ClusterContext:
    is_job_cluster: bool
    detection_method: str
    job_id: Optional[str]
    run_id: Optional[str]
    cluster_id: Optional[str]
    cluster_name: Optional[str]
    cluster_source: Optional[str]
    tags: Dict[str, str]
    env_db_is_job_cluster: Optional[bool]
    env_db_is_driver: Optional[bool]


def detect_cluster_context() -> ClusterContext:
    """
    Prefer Databricks context tags.
    Fallback to env vars if tags are missing.
    """
    tags = _safe_get_dbutils_context_tags()

    job_id = tags.get("jobId")
    run_id = tags.get("runId")
    cluster_id = tags.get("clusterId")
    cluster_name = tags.get("clusterName")
    cluster_source = tags.get("clusterSource")  # often: "UI" (interactive) or "JOB"

    env_job = _truthy_env("DB_IS_JOB_CLUSTER")
    env_driver = _truthy_env("DB_IS_DRIVER")

    # Primary detection: presence of jobId/runId OR clusterSource indicates JOB
    if job_id or run_id:
        return ClusterContext(
            is_job_cluster=True,
            detection_method="context_tags(jobId/runId)",
            job_id=job_id,
            run_id=run_id,
            cluster_id=cluster_id,
            cluster_name=cluster_name,
            cluster_source=cluster_source,
            tags=tags,
            env_db_is_job_cluster=env_job,
            env_db_is_driver=env_driver,
        )

    if (cluster_source or "").upper() == "JOB":
        return ClusterContext(
            is_job_cluster=True,
            detection_method="context_tags(clusterSource=JOB)",
            job_id=job_id,
            run_id=run_id,
            cluster_id=cluster_id,
            cluster_name=cluster_name,
            cluster_source=cluster_source,
            tags=tags,
            env_db_is_job_cluster=env_job,
            env_db_is_driver=env_driver,
        )

    # Fallback: env var if present
    if env_job is not None:
        return ClusterContext(
            is_job_cluster=bool(env_job),
            detection_method="env(DB_IS_JOB_CLUSTER)",
            job_id=job_id,
            run_id=run_id,
            cluster_id=cluster_id,
            cluster_name=cluster_name,
            cluster_source=cluster_source,
            tags=tags,
            env_db_is_job_cluster=env_job,
            env_db_is_driver=env_driver,
        )

    # Default: assume interactive/all-purpose
    return ClusterContext(
        is_job_cluster=False,
        detection_method="default(False)",
        job_id=job_id,
        run_id=run_id,
        cluster_id=cluster_id,
        cluster_name=cluster_name,
        cluster_source=cluster_source,
        tags=tags,
        env_db_is_job_cluster=env_job,
        env_db_is_driver=env_driver,
    )


# -------------------------
# 2) Log path + filename
# -------------------------
def utc_timestamp() -> str:
    # High-resolution UTC timestamp (safe + stable)
    return datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S_%f")


def choose_log_path(base_name: str, ctx: ClusterContext) -> str:
    """
    - Job compute: /databricks/driver/logs
    - Interactive/all-purpose: /app_logs
    """
    if ctx.is_job_cluster:
        base_dir = "/databricks/driver/logs"
    else:
        base_dir = "/app_logs"

    filename = f"{base_name}_{utc_timestamp()}.log"
    full_path = os.path.join(base_dir, filename)

    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    return full_path


# -------------------------
# 3) Formatter (Datadog-friendly levels)
# -------------------------
class DDStatusFormatter(logging.Formatter):
    """
    Adds dd_level field (WARN instead of WARNING) and formats log lines.
    """

    def format(self, record: logging.LogRecord) -> str:
        record.dd_level = "WARN" if record.levelname == "WARNING" else record.levelname
        return super().format(record)


# -------------------------
# 4) Logger setup
# -------------------------
def setup_tdv_logger(
    logger_name: str = "TDVIP_APP",
    log_file_base_name: str = "tdvip",
    enable_file_logging: bool = True,
) -> logging.Logger:
    ctx = detect_cluster_context()

    logger = logging.getLogger(logger_name)
    logger.setLevel(logging.INFO)
    logger.handlers = []
    logger.propagate = False

    fmt = "%(asctime)s %(dd_level)s %(name)s:%(lineno)d - %(message)s"
    formatter = DDStatusFormatter(fmt)
    formatter.default_time_format = "%Y-%m-%d %H:%M:%S"
    formatter.default_msec_format = "%s,%03d"

    # Stdout (should always be visible in driver logs)
    sh = logging.StreamHandler(sys.stdout)
    sh.setLevel(logging.INFO)
    sh.setFormatter(formatter)
    logger.addHandler(sh)

    # File handler (optional)
    log_path = None
    if enable_file_logging:
        try:
            log_path = choose_log_path(log_file_base_name, ctx)
            fh = logging.FileHandler(log_path)
            fh.setLevel(logging.INFO)
            fh.setFormatter(formatter)
            logger.addHandler(fh)
            logger.info(f"File logging enabled at {log_path}")
        except Exception as e:
            logger.warning(f"Could not enable file logging: {e}")

    # Diagnostic block (useful for Teams/Eilam)
    diag = {
        "is_job_cluster": ctx.is_job_cluster,
        "detection_method": ctx.detection_method,
        "jobId": ctx.job_id,
        "runId": ctx.run_id,
        "clusterId": ctx.cluster_id,
        "clusterName": ctx.cluster_name,
        "clusterSource": ctx.cluster_source,
        "env.DB_IS_JOB_CLUSTER": ctx.env_db_is_job_cluster,
        "env.DB_IS_DRIVER": ctx.env_db_is_driver,
        "log_path": log_path,
    }
    logger.info("Cluster context: " + json.dumps(diag, indent=2, sort_keys=True))

    return logger


# -------------------------
# 5) Example usage / test
# -------------------------
log = setup_tdv_logger(
    logger_name="TDVIP_APP",
    log_file_base_name="test31",   # your base filename (without .log)
    enable_file_logging=True,
)

log.info("Testing INFO level")
log.warning("Testing WARNING level")
log.error("Testing ERROR level")

# Optional: show recent file(s) if job cluster and writing to /databricks/driver/logs
# %sh
# ls -lah /databricks/driver/logs | tail -n 30
