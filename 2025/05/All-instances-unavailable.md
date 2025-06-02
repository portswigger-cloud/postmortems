# 27-05-2025 - All instances unavailable to BSEE SaaS customers

## Severity

**Critical** -  Complete outage impacting all customers.

## Security Related

**No**

## Impact

| Customers Impacted | Support Cases Raised | Customer Data Loss | Incident Duration |
| :----------------: | :------------------: | :----------------: | :---------------: |
|        All         |          3           |        None        |        1h         |


## Issue Summary
An upgrade to DAST 2025.5 failed due to the enterprise-server image not updating. This caused a version mismatch with the web component, resulting in failed startup of instances.

## Timeline

27-05-2025 13:44 – DAST 2025.5 rollout began.

27-05-2025 14:04 – Technical support reported CrashLoopBackOff.

27-05-2025 14:10 – Platform team began investigation.

27-05-2025 14:20 – Checksum mismatch in image deployment identified.

27-05-2025 14:31 – Fix deployed with correct image checksum.

27-05-2025 15:01 – All services restored.

## Root Cause
describeImageTags from AWS ECR returned unordered data. Automation consumed the wrong checksum, leading to a stale enterprise-server image being deployed.


## Resolution and recovery
Correct checksum was retrieved and deployed. Rollout validated in dev and prod before full release.

## Corrective and Preventative Measures 

1. Add alerting to verify instance health post-deploy.

2. Route failure alerts to Zoom for immediate triage.

3. Fail pipeline if rollout state is inconsistent.
   
4. Script updated to handle pagination and ensure correct checksum retrieval.
