# Wazuh Lab 127 — Linux Cron Persistence Investigation

## Overview

This lab investigates how Linux cron can be abused as a persistence mechanism and demonstrates how Wazuh File Integrity Monitoring (FIM) can detect changes to scheduled-task configuration files.

The investigation was performed on a Rocky Linux 9.8 virtual machine running Wazuh 4.13.1. Cronie provided the scheduling service, while Wazuh monitored selected cron directories and generated alerts through custom detection rules.

A controlled, harmless cron task was created, modified, executed, and removed to validate the detection workflow.

## Objectives

- Understand how cron can be used for legitimate scheduling and persistence.
- Establish a baseline of existing cron configuration.
- Monitor cron-related directories using Wazuh FIM.
- Develop custom rules to detect cron configuration creation, modification, and deletion.
- Correlate FIM alerts with cron execution logs.
- Validate detection using a controlled benign scheduled task.
- Document the investigation evidence, troubleshooting, and cleanup.

## Environment

| Component | Details |
|---|---|
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Virtualization | VMware |
| Architecture | x86-64 |
| Wazuh Version | 4.13.1 |
| Wazuh Revision | rc1 |
| Wazuh Role | Server |
| Cron Implementation | Cronie 1.5.7 |
| Cron Service | `crond` |
| FIM Component | `wazuh-syscheckd` |
| Analysis Component | `wazuh-analysisd` |

## Monitored Locations

The investigation configured real-time FIM monitoring for the following locations:

- `/etc/cron.d/`
- `/etc/cron.hourly/`
- `/etc/cron.daily/`
- `/etc/cron.weekly/`
- `/etc/cron.monthly/`
- `/var/spool/cron/`

The existing Wazuh configuration also monitored `/etc`, `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, and `/boot`.

## Detection Rules

| Rule ID | Detection | Severity | MITRE ATT&CK |
|---|---|---:|---|
| 100630 | New cron configuration created | 12 | T1053.003 |
| 100631 | Existing cron configuration modified | 12 | T1053.003 |
| 100632 | Cron configuration deleted | 10 | T1053.003 |

All three rules generated alerts during the controlled test.

## Investigation Summary

The initial cron baseline showed:

- The `crond` service was active and enabled.
- Cronie 1.5.7 was installed.
- `/etc/cron.d/` contained the existing `0hourly` file.
- `/etc/cron.hourly/` contained the existing `0anacron` script.
- The other inspected periodic cron directories contained no files.
- No crontab was configured for `root` or `abhinavmathur`.
- `/etc/cron.deny` existed and was empty; `/etc/cron.allow` was not found.

Wazuh FIM successfully monitored the configured cron directories. A test file named `lab127_heartbeat` was created under `/etc/cron.d/` with a benign command that appended timestamps to a temporary log.

Wazuh generated the following alerts:

- Rule `100630` detected creation of the cron configuration.
- Rule `100631` detected modification of the configuration.
- Rule `100632` detected deletion of the configuration.

Cron execution records showed the test commands being launched and completed. The timestamp output confirmed execution of the heartbeat task. The test configuration and temporary output directory were removed after validation.

## Evidence and Results

| Evidence | Result |
|---|---|
| Cron service status | Active and enabled |
| Cron configuration baseline | Captured |
| FIM monitoring | Real-time monitoring started for configured cron directories |
| Creation detection | Confirmed — Rule 100630 |
| Modification detection | Confirmed — Rule 100631 |
| Deletion detection | Confirmed — Rule 100632 |
| Cron execution | Confirmed through `/var/log/cron` |
| Heartbeat output | Timestamp entries observed |
| Test cleanup | Confirmed — test file and temporary directory absent |

## Troubleshooting

During rule configuration, `local_rules.xml` initially contained multiple top-level `<group>` elements, resulting in an XML parser error. The file was backed up and repaired into a single top-level group structure.

The XML validation and Wazuh analysis and Syscheck configuration tests completed without reported errors. The first Wazuh Manager restart attempt timed out, but a subsequent restart succeeded. The Manager, `wazuh-syscheckd`, and `wazuh-analysisd` were confirmed running.

Existing custom-ticket integration connection-refused errors and a duplicate Windows agent enrollment warning were also present in the Wazuh logs. These were separate from the cron FIM detection results.

## Conclusion

This lab demonstrated that Wazuh FIM and custom rules can detect the creation, modification, and deletion of monitored Linux cron configuration files. Cron execution logs and the heartbeat output provided additional evidence that the controlled scheduled task ran.

The results validate the detection workflow in a controlled environment. They do not establish that the system experienced malicious cron persistence or compromise.

**MITRE ATT&CK:** T1053.003 — Scheduled Task/Job: Cron
