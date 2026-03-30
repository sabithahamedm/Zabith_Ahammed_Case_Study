Secure-Pipe & Deception PoC
---------------------------

Project Overview
----------------

This repository contains the Infrastructure as Code (Terraform) and Logic (Python) for a resilient, automated security logging pipeline. The system ingestion follows the UDM (Universal Data Model) standard to ensure compatibility with modern SIEMs while providing an automated "Kill-Switch" for honeytoken compromises.

High-Level Architecture
-----------------------

The following diagram illustrates the flow from application log generation to immutable storage and automated incident response.

```mermaid
graph LR
    subgraph "Application Account (Production)"
        A[App / Lambda] -->|JSON Logs| B(CloudWatch Logs)
        B -->|Subscription Filter| C{Kinesis Firehose}
        C -->|Invoke| D[Python Transformer]
        D -->|UDM + €10k Tag| C
    end

    subgraph "Log Archive Account (Security)"
        C -->|Encrypted PUT| E[(S3 Log Vault)]
        E --- F[Object Lock: Compliance]
    end

    subgraph "Automated Response"
        G[Canarytoken Trigger] -->|EventBridge| H{Step Functions}
        H -->|Kill-Switch| I[Lambda: Revoke IAM]
        I -->|Alert| J[SNS / Slack]
    end

    style E fill:#f96,stroke:#333,stroke-width:2px
    style F fill:#dfd,stroke:#333,stroke-width:2px
    style I fill:#f66,stroke:#333,stroke-width:2px


Reliability: At-Least-Once Delivery & Retries
---------------------------------------------
In a security context, a missing log is a blind spot. This architecture ensures data integrity through three layers of protection:

Kinesis Firehose Buffering: Firehose stores data in a temporary buffer (configured for 5MB or 300s). If the destination (S3) is temporarily unavailable, Firehose automatically retries the delivery for up to 24 hours.

Lambda Synchronous Retries: If the "Transformer" Lambda fails due to a timeout or throttling, Firehose retries the invocation 3 times. If it still fails, the raw, unprocessed record is sent to a Special S3 Error Prefix (/errors/...) to prevent data loss.

S3 Object Lock: By using Compliance Mode, we ensure that once a log is written, it cannot be deleted or modified by any user, including the root account, for 90 days. This protects against "log scrubbing" during a breach.

Monitoring & Pipeline Health
----------------------------
To ensure the "Secure-Pipe" is healthy, we monitor four critical "Golden Signals" via CloudWatch.

1. The Success Rate (Integrity)
Metric: DeliveryToS3.Success

Why it matters: This must be 100%. Any drop indicates a permissions failure (KMS/IAM) or a service outage.

Alarm: Trigger a High-Priority Slack alert if success falls below 99% over a 5-minute window.

2. The Transformation Health (Logic)
Metric: ExecuteProcessing.Errors (Lambda)

Why it matters: High error rates mean the Python script is crashing—likely due to a change in the application's log format (Schema Drift).

Alarm: Trigger a Warning alert if errors > 5 in 15 minutes.

3. Data Freshness (Latency)
Metric: DeliveryToS3.DataFreshness

Why it matters: Measures the age of the oldest record in the pipe. If this climbs above 15 minutes, the pipe is "clogged," and the SOC is looking at stale data.

Alarm: Trigger an alert if Freshness > 900 seconds.

4. Throughput (Volume)
Metric: IncomingBytes / IncomingRecords

Why it matters: A sudden drop to 0 usually means the application or the CloudWatch Subscription Filter has been disconnected. A sudden 10x spike could indicate a DDoS attack or a recursive log loop.

Tech Stack
----------
IaC: Terraform v1.14+

Compute: AWS Lambda (Python 3.11)

Stream: Amazon Kinesis Data Firehose

Storage: Amazon S3 (Object Lock / SSE-KMS)

Analytics: Amazon Athena & AWS Glue

How to Use this Project
-----------------------
Deploy Storage: Run terraform apply in the /storage directory first to create the KMS keys and S3 vault.

Deploy Pipeline: Zip the transformer.py and run terraform apply in the /pipeline directory.

Verify: Use the provided Athena SQL queries to audit the land logs.
