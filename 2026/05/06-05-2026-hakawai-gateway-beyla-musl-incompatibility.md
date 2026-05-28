# 06-05-2026 - hakawai-gateway outage — Beyla agent injection incompatible with musl-libc base image

## Severity

**Critical** - Customer-facing gateway unable to serve traffic for ~9 hours.

## Security Related

**No**

## Impact

| Customers Impacted | Support Cases Raised | Customer Data Loss | Incident Duration |
| :----------------: | :------------------: | :----------------: | :---------------: |
| 490 unique IPs attempted to connect during the window | 0 | None | ~9h |

## Issue Summary

Both replicas of `hakawai-gateway` entered a crash loop after their pods were scheduled onto ARM64 nodes. A platform-managed observability agent (Grafana Beyla) running on every cluster node forcibly injects a Java agent into the gateway JVM; the agent's bundled native library is built against glibc, but the gateway's hardened base image is Alpine (musl-libc), so the library load fails. On ARM64 the failure path put the JVM into a ~75-second stall every time the pod started — long enough to fail the liveness probe and trigger a kill-and-restart loop.

The gateway was unable to serve traffic from ±01:00 to ±10:00 GMT — approximately 9 hours. Internal services that call gateway saw connection-refused errors during the window. 490 unique IP addresses attempted to connect to us during this time.

## Timeline

> (all times in GMT)

- **27-02-2026** – `hakawai-gateway` switched to the hardened Alpine base image (`dhi-eclipse-temurin:alpine`). The image bundles a diagnostic Java agent; the bug was latent until pods were placed on an ARM64 node.
- **06-05-2026 00:58** – Outage onset (per monitoring). Earlier cluster events for the failing pods were not retained by the time engineering investigated.
- **06-05-2026 ~08:50** – Outage noticed by the team.
- **06-05-2026 ~09:30** – Engineering identified the root cause from gateway pod logs.
- **06-05-2026 09:43** – Fix merged: pin `hakawai-gateway` to amd64 nodes.
- **06-05-2026 09:55** – Fix rolled out by the standard deployment pipeline.
- **06-05-2026 10:00** – Deployment regained minimum availability and gateway returned to steady state.

## Root Cause

A `grafana-beyla` DaemonSet runs on every node in the cluster with admin privileges. It copies `obi-java-agent.jar` into application pods' `/tmp` directories and invokes a tool that attaches the jar to the running JVM as a dynamic Java agent. The jar contains a native library `libobijni.so` for both amd64 and arm64, but the library is built against glibc, while the gateway's hardened base image is Alpine — which uses musl-libc. `dlopen()` rejects the library on a musl-libc system. The agent's error-handling code then reports the failure as a wrong-architecture problem (`unsupported relocation type 7 ... can't load AMD 64 .so on a AARCH64 platform`); that diagnostic is misleading — the actual problem is the libc mismatch, not the architecture.

After the failed load, the JVM enters a ~75-second period of complete unresponsiveness on ARM64 during which application threads, background tasks, and incoming requests all halt. The gateway's liveness probe allows only 30 seconds of unresponsiveness before the kubelet kills the pod, so each pod is killed mid-stall and restarted indefinitely. On amd64 the same library load also fails, but the resulting failure is benign or brief enough that the JVM does not stall past the liveness budget — exactly why is to be confirmed.

The defect had been latent since the gateway switched to the Alpine base image in February. A single pod landing on ARM64 would have crash-looped one replica while the other kept serving (degraded but not user-visible); the 6 May outage happened because both replicas landed on ARM64 at the same time.

Other JVM services using the same base image logged the same library-load failure but did not crash; their post-startup workload is lighter and stayed inside the liveness budget.

## Resolution and Recovery

Engineering pinned `hakawai-gateway` to amd64 nodes by adding a node selector to the deployment. The fix went out through the standard deployment pipeline and pods rescheduled cleanly onto amd64. Steady-state CPU and request handling returned to normal immediately, and downstream services stopped seeing connection-refused errors. This is a workaround that dodges the symptom on ARM64; the underlying Beyla / musl-libc incompatibility remains.

## Corrective and Preventative Measures

1. Disable Beyla injection on `hakawai-gateway` (and other Alpine-based JVM services) until someone is actively consuming the metrics it is collecting. This is the proximate root-cause fix; the amd64 pin can be removed afterwards.
2. Apply the amd64 pin in the meantime to other Alpine-based services that are crash-looping with the same Beyla error. `hakawai-recorded-login-generator` is the immediate candidate.
3. Engage Beyla maintainers about musl-libc support — either ship a musl-built native library, or have the agent detect a musl host and skip injection cleanly. The current behaviour (force-attach a glibc binary, then misreport the failure as an architecture problem) is a defect.
4. Platform team: review the design of cluster-wide DaemonSets that forcibly inject and attach code into application pods. A failure mode in such an agent should not be able to take a customer-facing service down. At minimum: opt-in injection, and a guardrail that prevents workloads from being targeted that have not been certified compatible.
5. Investigate why amd64 + musl-libc tolerates the failed library load while ARM64 + musl-libc produces a 75-second JVM stall. The answer informs what other workloads might be at risk.
6. Add CI coverage that boots the produced container image on an ARM64 runner and verifies the service passes its startup and liveness probes within budget.
7. Add an alert on the JVM thread-starvation log signature — the single most diagnostic signal during this incident.
8. Revert the temporary CVE severity-gate relaxation that was introduced alongside the fix to unblock the deploy, once the underlying dependency is bumped.
