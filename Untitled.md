databricks interview questions


- [ ] Difference between all-purpose clusters and job clusters? Advantages of each?
- [ ] What is the purpose of Unity Catalog?
- [ ] What is the difference between:
        - [ ] Dataset
        - [ ] DataFrame
        - [ ] RDD
- [ ] How do you integrate Kafka with Databricks Structured Streaming?
        - [ ] Discuss: offset management, committing offsets, throughput.
- [ ] How do you handle late arriving data in Structured Streaming?
        - [ ] Explain: watermarking, state cleanup, upsert to Delta.
- [ ] How do you secure secrets in Databricks?
- [ ] Key Vault integration, databricks-backed secrets, ACLs
- [ ] Difference between Instance Pools vs Cluster Pools.
- [ ] (You asked earlier)
        - [ ] instance pool = VM warm-up
        - [ ] cluster pool = reuse pools of pre-configured clusters
        - [ ] Difference between Jobs and Workflows.

- [ ] How does Autoloader (cloudFiles) work internally?
        - [ ] Explain:
        - [ ] Notification mode (Event Grid, SQS)
        - [ ] Directory listing mode
        - [ ] Checkpointing
        - [ ] File tracking logs
- [ ] How does Serverless SQL Warehouse differ from classic?
- [ ] What is the role of the control plane vs data plane?
- [ ] How do you enforce table-level and column-level permissions?
- [ ] Difference between groups, service principals, and PAT tokens
- [ ] How to integrate with Azure AD or AWS IAM?
- [ ] How do you restrict access to:
    - [ ] a cluster
    - [ ] a job
    - [ ] a volume
    - [ ] a schema/table/view
- [ ] How to securely store secrets? (Databricks Secrets Backend, AAD Key Vault)
Design questions
- [ ] Design a real-time CDC pipeline using Auto Loader + Delta Lake + SQL Warehouse.
- [ ] Design an SCD Type-2 pipeline using Databricks.
- [ ] Architect a cross-tenant ingestion layer with Unity Catalog.
    - [ ] How would you ingest 10TB/day with 5-minute SLA?
    - [ ] How do you structure Bronze → Silver → Gold layers?
- [ ] How do you architect pipelines to avoid downtime during schema changes?
- [ ] What is a Task? Types of tasks supported?
- [ ] How do you implement error handling and retry policies?
- [ ] How do you pass data between tasks? (Task Values, DBFS, Delta tables)

- [ ] How does Databricks handle schema evolution & enforcement?
	Modes: add column, rename, drop, constraints.