import os
import sys
import logging
from logging.handlers import RotatingFileHandler
from datetime import datetime

# ---------------------------------------------------
# 1️⃣ Detect cluster type
# ---------------------------------------------------

def detect_cluster_type():
    is_job_cluster = os.environ.get("DB_IS_JOB_CLUSTER", "").upper() == "TRUE"

    cluster_type_tag = None
    try:
        cluster_type_tag = spark.conf.get(
            "spark.databricks.clusterUsageTags.clusterType"
        )
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

# ---------------------------------------------------
# 2️⃣ Setup logger
# ---------------------------------------------------

def setup_tdvip_logger(context):
    logger = logging.getLogger("TDVIP_APP")
    logger.setLevel(logging.INFO)
    logger.handlers = []
    logger.propagate = False

    formatter = logging.Formatter(
        "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
    )

    # Always log to STDOUT
    stdout_handler = logging.StreamHandler(sys.stdout)
    stdout_handler.setFormatter(formatter)
    logger.addHandler(stdout_handler)

    # If job cluster, also log to driver logs path
    if context["is_job_cluster"]:
        try:
            log_path = "/databricks/driver/logs/tdvip_app.log"

            file_handler = RotatingFileHandler(
                log_path,
                maxBytes=10_000_000,
                backupCount=3
            )
            file_handler.setFormatter(formatter)
            logger.addHandler(file_handler)

            logger.info(f"Job cluster detected. File logging enabled at {log_path}")

        except Exception as e:
            logger.warning(
                f"Could not enable file logging in driver logs directory: {e}"
            )
    else:
        logger.info("Standard (interactive) cluster detected.")

    logger.info(f"Cluster context: {context}")
    return logger


log = setup_tdvip_logger(ctx)

# ---------------------------------------------------
# 3️⃣ Example usage
# ---------------------------------------------------

log.info("TDVIP logging initialized.")
log.info("Testing INFO level")
log.warning("Testing WARNING level")
log.error("Testing ERROR level")
