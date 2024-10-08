# 17-09-2024 - Github action-runners failed which caused a six hour code deployment issues across the shared-platform  

## Severity & Impact
Critical - multiple users unable to work

## Issue Summary
End users working predominately on the Burp-AI product team couldn't run their pipelines to deploy code changes due to the action runners not being available.

## Timeline
* 17-09-2024 08:30 - Users started to complain due to not being able to run their code pipelines.
* 17-09-2024 09:30 - In the k9 console the action-runner pods where failing to start and it was possible to grab the logs of a failing pod before it shut down and identify the error from the error message.
* 17-09-2024 11:00 -  The cause was identified as a github update of their actions-runners container from 2.3.17.0 to 2.3.19.1 where the old version was no longer supported or backwards compatible. 
* 17-09-2024 14:00 - In order to solve the issue we updated our references to the new container and rolled it out to the platform-dev and platform-prod environments.

## Root Cause
It is believed that the root cause of the issue was a new release of the github actions-runner.

## Resolution and recovery
2.3.19.1 was rolled out as soon as it was identified and resolved the issues.

## Corrective and Preventative Measures
* We don't want to pin ourself to certain versions, but we will be able to remediate and fix forward within a much smaller time frame should this error occur again (1hr). We could also create an alert for this error and have a ticket in the backlog for further remedial actions.

## Ticket trail
* https://portswigger.atlassian.net/issues/SD-458
* https://portswigger.atlassian.net/issues/TECH-580
* https://portswigger.atlassian.net/issues/TECH-581
