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

On 28 November 2025, AWS Load Balancer Controller was upgraded from v1.14.1 to v1.16.0. A change was made to the Helm chart in v1.15.0 that changed how certificates are managed: https://github.com/kubernetes-sigs/aws-load-balancer-controller/pull/4359 which resulted in an orphaned TLS secret.

Because the orphaned TLS secret was owned by a Certificate object that no longer existed, cert-manager could not renew it. The  certificate contained withing the orphaned TLS secret expired on 8 January 2026, causing webhook validation failures that prevented the load balancer controller from functioning.

The webhook certificate is required for load balancer target group configuration. Its expiration blocked the AWS Load Balancer Controller from updating AWS Target Group configurations eventually resulting in zero healthy targets.

## Resolution and Recovery

Platform Team identified expired webhook certificates across affected clusters. Resolution involved:

1. Deleting orphaned `aws-load-balancer-tls` secrets in US and Canada clusters
2. Redeploying AWS Load Balancer Controller configuration via Helm
3. Verifying certificate regeneration and webhook functionality

All services were restored once new certificates were generated and webhooks became operational.

## Corrective and Preventative Measures

1. Audit all clusters for orphaned secrets without owning Certificate resources and resolve.
2. Create high-level application access alerting for each SaaS cluster from an external probe.
