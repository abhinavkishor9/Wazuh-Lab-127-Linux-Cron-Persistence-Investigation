
# Investigation Timeline — Linux Cron Persistence

**Date:** 4 November 2026  
**Host:** `localhost.localdomain`  
**Platform:** Rocky Linux 9.8  
**Wazuh:** 4.13.1

| Time (IST) | Activity | Evidence / Result |
|---|---|---|
| 07:06:47 | Cron service startup | `crond` started successfully. |
| 07:17:20 | Existing user crontabs inspected | Cron log recorded listing requests for `root` and `abhinavmathur`. Neither account had a crontab. |
| 07:17 onward | Cron baseline collected | Existing files, permissions, ownership, system crontab, and cron execution logs were inspected. |
| 07:34:48 | Wazuh rule XML backup and repair | A timestamped backup of `local_rules.xml` was created, and multiple top-level groups were consolidated into one top-level group. |
| 07:36:52 | Wazuh Syscheck shutdown | Syscheck logged receipt of a shutdown signal during the Manager restart sequence. |
| 07:37:41 | Wazuh Syscheck startup | Syscheck started and registered the configured cron directories for monitoring. |
| 07:38:54 | FIM initialization completed | The initial FIM scan ended and the real-time monitoring module started. |
| 07:43:20 | Test cron configuration created | Wazuh Rule `100630` detected creation of `/etc/cron.d/lab127_heartbeat`. |
| 07:44:01 | First heartbeat execution | Cron logged execution of the timestamp command. The output file recorded the timestamp. |
| 07:45:01 | Heartbeat execution | Cron logged another execution, and the output file recorded another timestamp. |
| 07:46:01 | Heartbeat execution | Cron logged another execution, and the output file recorded another timestamp. |
| 07:47:01 | Cron configuration modified | A hostname command was added. Wazuh Rule `100631` detected the modification. Cron logged a reload of the configuration and execution of both commands. |
| 07:48:02 | Scheduled commands executed | Cron logged execution and completion of the hostname and timestamp commands. |
| After validation | Test cron configuration deleted | Wazuh Rule `100632` detected deletion of `/etc/cron.d/lab127_heartbeat`. |
| After deletion | Temporary test artifacts removed | Verification showed that the cron configuration file and `/var/tmp/lab127` directory no longer existed. |

## Timeline Assessment

The timeline demonstrates the complete controlled test lifecycle:

1. Existing cron configuration was inventoried.
2. Wazuh FIM monitoring was activated for the selected cron directories.
3. A benign scheduled task was created and detected.
4. The task was modified and the modification was detected.
5. Cron execution records confirmed that the scheduled commands ran.
6. The test configuration was deleted and the deletion was detected.
7. The temporary test artifacts were removed.

The timeline documents a controlled detection exercise. It does not represent a confirmed malicious incident or attacker activity.
