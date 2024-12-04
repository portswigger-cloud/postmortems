# 29-11-2024 - Customers unable to start new PortSwigger Hosted Scanning Machine scans

## Severity

**High** - Significant degradation or partial outage affecting many customers or resulting in potential security concerns.

## Security Related

**No**

## Impact

| Customers Impacted | Support Cases Raised | Customer Data Loss | Incident Duration |
|:---:|:---:|:---:|:---:|
| All | 2 | None | 15h |

## Issue Summary

When a SaaS Burp Suite Enterprise Instance attempted to start a scan on a PortSwigger Hosted Scanning Machine the scan would move into the queued state and not start.

CI driven scans and Customer Hosted Scanning Machine were **not** affected by this incident.

## Timeline

- 28-11-2024 21:00 - The metrics show that after this time no new scans where scheduled onto PortSwigger Hosted Scanning Machines.
- 29-11-2024 13:00 - Two customers raised support requests about not being able to start scans from their SaaS instances.
- 29-11-2024 13:15 - A major incident was declared and the Major Incident Management process is initiated. This included the start of the cross team investigation of the issue and the 
- 29-11-2024 13:30 - The [status page](https://status.portswigger.net/) was updated to show that there was a degradation in service.
- 29-11-2024 13:49 - The issue was resolve by draining the Kubernetes node that all the scan controllers had consolidated onto.
- 29-11-2024 14:03 - Resolution message posted to the [status page](https://status.portswigger.net/).

## Root Cause

Post incident investigations found that the Kubernetes EC2 node were all the scan controllers had consolidated became unhealthy at around 21:00 on the 28-11-2024, this resulted in the scan controllers being unable to resolve the DNS entry for the associated Burp Suite Enterprise instance.

This resulted in the scan controller not querying the scan job queue on the Burp Suite Enterprise instance and therefore it was unaware that there were scans to be dispatched.

## Corrective and Preventative Measures 

1. Introduce better logs and metric so that we are aware of queue scans and disconnected scan controllers quicker.
1. We will stop the scan controllers consolidating on one Kubernetes node reducing the customers impacted of an unhealthy node.
1. Create alerting and dashboard to surface any scan controller issues more quickly.