# Databricks notebook source
# TDVIP logger that works on BOTH:
# - Job compute: writes to /databricks/driver/logs/<file>.log (Datadog already tails it)
# - All-purpose: writes to /app_logs/<file>.log (Datadog tails /app_logs/*/*.log per your config)
#
# Also logs to STDOUT (so you always see logs in notebook/job output)
# Uses JSON lines so Datadog can reliably extract severity (INFO/WARN/ERROR)

import os
import sys
import json
import logging
from datetime import datetime
from logging.handlers import RotatingFileHandler

# -----------------------------
# 1) Detect cluster type
# -----------------------------
def detect_cluster_type():
    # If env var exists, use it
    is_job_cluster = os.environ.get("DB_IS_JOB_CLUSTER", "").upper() == "TRUE"

    # Try spark tag (best effort)
    cluster_type_tag = None
    try:
        cluster_type_tag = spark.conf.get("spark.databricks.clusterUsageTags.clusterType")
        if cluster_type_tag and cluster_type_tag.upper() == "JOB":
            is_job_cluster = True
    except Exception:
        pass

    return {
        "is_job_cluster": is_job_cluster,
        "cluster_type_tag": cluster_type_tag,
        "cluster_id": os.environ.get("DATABRICKS_CLUSTER_ID"),
        "job_id": os.environ.get("DATABRICKS_JOB_ID"),
        "run_id": os.environ.get("DATABRICKS_RUN_ID"),
    }

ctx = detect_cluster_type()

# -----------------------------
# 2) Choose a log path based on cluster type
# -----------------------------
def choose_log_path(log_file: str, context: dict) -> str:
    # Ensure .log suffix
    if not log_file.endswith(".log"):
        log_file = f"{log_file}.log"

    if context.get("is_job_cluster"):
        # Job compute path (Datadog is configured to tail /databricks/driver/logs/*.log)
        base_path = "/databricks/driver/logs"
    else:
        # All-purpose path (your init script config tails /app_logs/*/*.log)
        # Put team/service under a folder to avoid one huge shared directory
        base_path = "/app_logs/tdvip"

    full_path = f"{base_path}/{log_file}"

    # Create dirs only where we’re allowed/need to:
    # - /app_logs/... might not exist on all-purpose, so create it
    # - /databricks/driver/logs should already exist; creating it is harmless if permitted
    try:
        os.makedirs(os.path.dirname(full_path), exist_ok=True)
    except Exception:
        # If we can't create it, we'll still try file handler; if that fails, we fallback to stdout only
        pass

    return full_path

# -----------------------------
# 3) JSON formatter for Datadog-friendly parsing
# -----------------------------
class DDJsonFormatter(logging.Formatter):
    def __init__(self, context: dict):
        super().__init__()
        self.context = context or {}

    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": datetime.utcnow().isoformat() + "Z",
            "level": record.levelname,          # Datadog can map this to status
            "logger": record.name,
            "msg": record.getMessage(),
            # Helpful context for filtering in Datadog
            "is_job_cluster": self.context.get("is_job_cluster"),
            "cluster_type_tag": self.context.get("cluster_type_tag"),
            "cluster_id": self.context.get("cluster_id"),
            "job_id": self.context.get("job_id"),
            "run_id": self.context.get("run_id"),
        }
        return json.dumps(payload, ensure_ascii=False)

# -----------------------------
# 4) Create logger (stdout + rotating file)
# -----------------------------
def setup_tdvip_logger(context: dict, log_file_name: str = "tdvip_app") -> logging.Logger:
    logger = logging.getLogger("TDVIP_APP")
    logger.setLevel(logging.INFO)
    logger.handlers = []
    logger.propagate = False

    formatter = DDJsonFormatter(context)

    # Always log to STDOUT (visible in notebook/job output)
    sh = logging.StreamHandler(sys.stdout)
    sh.setFormatter(formatter)
    logger.addHandler(sh)

    # File logging (for Datadog tailing)
    log_path = choose_log_path(log_file_name, context)

    try:
        fh = RotatingFileHandler(
            log_path,
            maxBytes=10_000_000,   # 10 MB per file
            backupCount=3          # keeps 3 rotated files
        )
        fh.setFormatter(formatter)
        logger.addHandler(fh)

        logger.info(f"File logging enabled at {log_path}")
    except Exception as e:
        logger.warning(f"File logging NOT enabled (path={log_path}): {e}")

    logger.info(f"Cluster context: {context}")
    return logger

# -----------------------------
# 5) Example usage
# -----------------------------
# Change this per team/subteam/service to keep logs separate
# e.g., "vpda_ingestion", "rams_quality", "tdvip_bronze"
log = setup_tdvip_logger(ctx, log_file_name="tdvip_test2")

log.info("TDVIP logging initialized.")
log.info("Testing INFO level")
log.warning("Testing WARNING level")
log.error("Testing ERROR level")
