# Wazuh Lab 127 — Linux Cron Persistence Investigation
## Overview
Cron is a time-based job scheduler used by Linux systems to execute commands and scripts automatically at specified intervals. System administrators commonly use it for routine maintenance, log rotation, backups, monitoring, and other scheduled activities.

However, cron can also be abused as a persistence mechanism. An attacker who gains sufficient access to a Linux system may create or modify scheduled tasks to repeatedly execute an unauthorised command or script. This can allow unwanted activity to continue after the initial access has ended or after a system reboot.

Cron-related persistence can be configured through several locations, including:

- `/etc/crontab` — system-wide scheduled tasks.
- `/etc/cron.d/` — additional system-wide cron job definitions.
- `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/` — periodic maintenance scripts.
- `/var/spool/cron/` — individual users' crontab files.

A cron configuration change is not automatically malicious. Legitimate software packages and system administrators may create or update scheduled tasks. An investigation must therefore establish what changed, which file was affected, who owns it, what command is configured, and whether the scheduled command actually executed.

### Role of Wazuh

Wazuh File Integrity Monitoring (FIM) can monitor cron-related directories and detect file creation, modification, and deletion. Custom Wazuh rules can classify these events as potential cron persistence activity and map them to MITRE ATT&CK technique **T1053.003 — Scheduled Task/Job: Cron**.

FIM evidence can establish that a monitored file changed. Cron execution logs provide additional context about whether the scheduler ran a command. Reviewing both sources helps distinguish a configuration change from confirmed execution.

### Investigation Approach

This lab uses a controlled, harmless cron task to demonstrate the detection lifecycle:

1. Establish the existing cron configuration and scheduled-task baseline.
2. Configure Wazuh FIM to monitor relevant cron directories.
3. Create custom rules for cron configuration creation, modification, and deletion.
4. Add a benign scheduled task that writes timestamps to a temporary log.
5. Confirm that Wazuh detects the new cron configuration.
6. Modify the task and examine the corresponding FIM alert.
7. Correlate cron execution records with the temporary output.
8. Remove the test task and confirm that Wazuh detects its deletion.

### Security Significance

Cron monitoring helps security teams identify unexpected scheduled commands, suspicious changes to system-wide task definitions, and potential persistence mechanisms. Detection should be followed by contextual analysis rather than an automatic conclusion of compromise.

The most reliable assessment combines file integrity events, file ownership and permissions, scheduled command content, execution logs, and relevant system activity.

This lab investigates how Linux cron can be abused as a persistence mechanism and demonstrates how Wazuh File Integrity Monitoring (FIM) can detect changes to scheduled-task configuration files.

The investigation was performed on a Rocky Linux 9.8 virtual machine running Wazuh 4.13.1. Cronie provided the scheduling service, while Wazuh monitored selected cron directories and generated alerts through custom detection rules.

A controlled, harmless cron task was created, modified, executed, and removed to validate the detection workflow.

## Lab Objectives 

The objectives of this lab are to:

1. **Understand cron-based persistence:** Examine how Linux cron can be used to execute commands automatically and how unauthorised scheduled tasks may provide an attacker with a persistence mechanism.

2. **Identify cron configuration locations:** Explore system-wide cron definitions, periodic task directories, and user-specific crontab locations to understand where scheduled tasks can be configured.

3. **Establish a cron baseline:** Inspect existing scheduled tasks, file ownership, permissions, and cron service activity to identify the system's normal configuration before testing.

4. **Configure File Integrity Monitoring:** Use Wazuh FIM to monitor important cron directories and identify changes to scheduled-task configuration files.

5. **Develop custom detection rules:** Create and validate Wazuh rules to identify the creation, modification, and deletion of cron configuration files.

6. **Map detected activity to MITRE ATT&CK:** Associate the observed cron-related activity with **T1053.003 — Scheduled Task/Job: Cron**.

7. **Correlate configuration changes with execution evidence:** Compare Wazuh FIM alerts with cron service logs and command output to determine whether a scheduled task was actually executed.

8. **Validate the detection lifecycle:** Perform controlled creation, modification, execution, and deletion of a harmless cron task to evaluate the detection workflow.

9. **Distinguish suspicious changes from malicious activity:** Assess file content, ownership, permissions, execution records, and surrounding context before classifying a cron change as potentially malicious.

10. **Verify cleanup and document findings:** Remove temporary test configurations and output files, confirm their removal, and record detection results, limitations, and troubleshooting observations.

## Lab Scenario

A Linux server may be vulnerable to persistence through scheduled tasks when an unauthorised user gains sufficient access to create or modify cron jobs. By configuring a command to run at regular intervals, an attacker could maintain recurring access or repeatedly execute unwanted activity without requiring continuous interaction with the system.

In this scenario, a SOC analyst is responsible for investigating potential cron-based persistence on a Rocky Linux 9.8 server monitored by Wazuh. The objective is to determine whether changes to scheduled-task configuration can be detected and whether those changes can be correlated with actual command execution.

The analyst begins by establishing the existing cron baseline, reviewing system-wide and user-specific scheduling locations, checking file ownership and permissions, and examining the status of the `crond` service. This provides a reference for distinguishing expected system configuration from changes introduced during the investigation.

Wazuh File Integrity Monitoring is then configured to monitor important cron-related directories. Custom detection rules are introduced to identify three types of configuration changes:

- Creation of a new cron configuration file.
- Modification of an existing scheduled-task definition.
- Deletion of a monitored cron configuration file.

To validate the detection workflow without introducing harmful activity, the analyst creates a temporary cron configuration under `/etc/cron.d/`. The scheduled task runs a harmless command that appends timestamps to a temporary log file. The configuration is subsequently modified to include another benign command, allowing the analyst to observe how Wazuh handles changes to an existing scheduled task.

The investigation correlates the resulting FIM alerts with records in `/var/log/cron` and the temporary command output. This helps establish whether the configuration was merely changed or whether the scheduled commands were actually executed.

Finally, the analyst removes the temporary cron configuration and its output files, then verifies that Wazuh detects the deletion and that the test artifacts are no longer present.

The exercise demonstrates how cron configuration monitoring, file integrity alerts, and scheduler execution records can be combined to investigate a potential persistence mechanism. However, the controlled changes are not evidence of an actual attack. Any real-world alert would require further investigation of the command, file ownership, permissions, initiating user, and surrounding system activity before malicious intent could be established.

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

