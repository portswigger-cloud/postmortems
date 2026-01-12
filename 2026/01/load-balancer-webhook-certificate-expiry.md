# 12-01-2026 - BSEE SaaS instances unavailable in US and Canada regions

## Severity

**Critical** - Complete outage affecting US and Canada region customers.

## Security Related

**No**

## Impact

| Customers Impacted | Support Cases Raised | Customer Data Loss | Incident Duration |
| :----------------: | :------------------: | :----------------: | :---------------: |
|  US/Canada regions |          2           |        None        |       ~48m        |

## Issue Summary

An expired TLS certificate for the AWS Load Balancer Controller webhook caused all BSEE SaaS instances in US and Canada regions to become unavailable. Customers were unable to access the Web UI or start scans. The issue stemmed from an orphaned certificate that was not renewed after a Helm chart upgrade in November 2025.

## Timeline

- **08-01-2026** – AWS Load Balancer Controller webhook certificate expired
- **11-01-2026 10:00 UTC** – Webhook failures began impacting service availability
- **12-01-2026 09:52 UTC** – Issue reported by customers via support
- **12-01-2026 10:00 UTC** – Platform Team began investigation
- **12-01-2026 10:40 UTC** – Resolution deployed, services restored

## Root Cause

On 28 November 2025, AWS Load Balancer Controller was upgraded from v1.14.1 to v1.16.0. This version changed how certificates are managed, removing the `aws-load-balancer-serving-cert` Certificate resource while leaving the associated secret in place.

Without an active Certificate resource, cert-manager did not renew the webhook TLS certificate. The orphaned certificate expired on 8 January 2026, causing webhook validation failures that prevented the load balancer controller from functioning.

The webhook certificate is required for pod admission and load balancer configuration. Its expiration blocked:
- Web UI access
- Scan initiation
- Service availability callbacks

## Resolution and Recovery

Platform Team identified expired webhook certificates across affected clusters. Resolution involved:

1. Deleting orphaned `aws-load-balancer-tls` secrets in US and Canada clusters
2. Redeploying AWS Load Balancer Controller configuration via Helm
3. Verifying certificate regeneration and webhook functionality

All services were restored once new certificates were generated and webhooks became operational.

## Corrective and Preventative Measures

1. Implement monitoring to alert on certificate provisioning/expiration
2. Add automated validation of webhook certificate validity in cluster health checks
3. Audit all clusters for orphaned secrets without owning Certificate resources
