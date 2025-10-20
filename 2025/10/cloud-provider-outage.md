# 20-10-2025 - Cloud Provider Outage Impacting DAST SaaS

## Severity

**Critical** – Due to an outage affecting a major cloud provider, new nodes within our compute clusters were unable to start and become ready for approximately two hours.


## Security Related

**No**

## Impact

| Customers Impacted | Support Cases Raised | Customer Data Loss | Incident Duration |
| :----------------: | :------------------: | :----------------: | :---------------: |
|      Saas Customers     |        Not Known           |        None        |       ~2h        |

## Issue Summary

During this period:
* New scans were unable to start
* Provisioning of new SaaS instances and databases failed
* Existing scans and running services were unaffected, and no customer data was lost during the incident.

## Timeline

- **09:15 GMT** – Monitoring detected failures in scan provisioning
- **09:30 GMT** – Major incident declared
- **09:30 GMT** – Root cause traced to upstream cloud provider issues
- **10:40 GMT** – Provider systems recovered
- **10:45 GMT** – All services confirmed operational
- **11:51 GMT** – Incident formally closed

## Root Cause

An outage in AWS services, including DynamoDB, Route 53, and API Gateway, caused a ripple effect that disrupted dependent services such as `quay.io` and `docker.io`. Our infrastructure relies on these registries to pull container images during autoscaling. The inability to fetch required images prevented node readiness, blocking new scan jobs and SaaS provisioning.

## Resolution and Recovery

Service automatically recovered once the cloud provider’s systems returned to normal operation. We have verified that all affected systems are now fully functional.

We are investigating ways to improve the resilience of our infrastructure to minimize the impact of upstream provider issues in the future.

## Corrective and Preventative Measures

Evaluate use of a **pull-through cache (e.g., Harbor)** to reduce reliance on external registries  