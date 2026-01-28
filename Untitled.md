import logging
import time

print("DATADOG_TEST: notebook started")

logger = logging.getLogger("datadog-test")
logger.setLevel(logging.INFO)

logger.info("DATADOG_TEST: info log from job cluster")
logger.warning("DATADOG_TEST: warning log from job cluster")
logger.error("DATADOG_TEST: error log from job cluster")

time.sleep(30)

print("DATADOG_TEST: notebook finished")
