# Investigation Notes

## 1. Host and Service Baseline

The system was identified as a VMware virtual machine running Rocky Linux 9.8 with kernel `5.14.0-687.31.1.el9_8.x86_64`.

Wazuh reported version `v4.13.1`, revision `rc1`, and server role. The Wazuh Manager was active, and both `wazuh-syscheckd` and `wazuh-analysisd` were running.

Cronie version `1.5.7-16.el9.x86_64` was installed. The `crond` service was active and enabled.

### Existing Cron Configuration

The baseline inspection identified:

| Location | Existing contents |
|---|---|
| `/etc/crontab` | Default system crontab configuration; no active job entries observed |
| `/etc/cron.d/` | `0hourly` |
| `/etc/cron.hourly/` | `0anacron` |
| `/etc/cron.daily/` | No files observed |
| `/etc/cron.weekly/` | No files observed |
| `/etc/cron.monthly/` | No files observed |
| `/var/spool/cron/` | No files found by the baseline inventory |

The `0hourly` file had permissions `644` and ownership `root:root`. The `0anacron` script had permissions `755` and ownership `root:root`.

No crontab was configured for either `root` or `abhinavmathur` at the time of inspection.

`/etc/cron.deny` existed and was empty. `/etc/cron.allow` was not found.

These observations describe the inspected configuration at the time of the baseline. They do not establish that no other scheduling mechanism or persistence method existed on the host.

## 2. Wazuh FIM Configuration

The Wazuh Syscheck configuration was updated to monitor the following cron-related paths in real time:

- `/etc/cron.d`
- `/etc/cron.hourly`
- `/etc/cron.daily`
- `/etc/cron.weekly`
- `/etc/cron.monthly`
- `/var/spool/cron`

The existing FIM configuration also included `/etc`, `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, and `/boot`.

After the Manager restart, Wazuh logs confirmed that the cron directories were registered for real-time monitoring. The logs also showed the initial FIM scan completing and real-time monitoring starting.

## 3. Custom Detection Rules

Three custom rules were configured in `local_rules.xml`.

| Rule ID | Description | Level | Event |
|---|---|---:|---|
| 100630 | Possible cron persistence: new scheduled-task configuration created | 12 | Added |
| 100631 | Possible cron persistence: scheduled-task configuration modified | 12 | Modified |
| 100632 | Possible cron persistence: scheduled-task configuration deleted | 10 | Deleted |

The rules mapped the activity to MITRE ATT&CK technique `T1053.003` — Scheduled Task/Job: Cron.

The rule descriptions use the term "possible cron persistence" because a file change alone does not prove malicious intent.

## 4. Controlled Test

A temporary directory was created at:

`/var/tmp/lab127`

A test cron configuration was created at:

`/etc/cron.d/lab127_heartbeat`

The initial scheduled command was:

`* * * * * root /usr/bin/date -Is >> /var/tmp/lab127/heartbeat.log 2>&1`

The file was owned by `root:root` and assigned permissions `0644`.

The command was deliberately benign. It appended an ISO-formatted timestamp to a temporary output file once per minute.

### Creation Detection

Wazuh generated an alert with:

- Rule ID: `100630`
- Level: `12`
- File: `/etc/cron.d/lab127_heartbeat`
- Event: `added`
- FIM mode: `realtime`
- Owner: `root`
- Group: `root`
- Permissions: `rw-r--r--`

The alert included file metadata and hashes, providing evidence of the newly created file.

### Execution Evidence

The heartbeat output contained the following timestamps:

- `2026-11-04T07:44:01+05:30`
- `2026-11-04T07:45:01+05:30`
- `2026-11-04T07:46:01+05:30`
- `2026-11-04T07:47:01+05:30`

The cron log recorded the corresponding command executions and completions. This confirmed that the controlled heartbeat task was executed by cron.

### Modification Detection

The cron configuration was updated to include a second benign scheduled command:

`* * * * * root /usr/bin/hostname >> /var/tmp/lab127/hostname.log 2>&1`

The Wazuh alert screenshot showed:

- Rule ID: `100631`
- Level: `12`
- File: `/etc/cron.d/lab127_heartbeat`
- Event: `modified`
- MITRE ATT&CK: `T1053.003`

The cron log recorded the reload of `/etc/cron.d/lab127_heartbeat` and execution of both the hostname and timestamp commands.

The available output confirms the heartbeat command's results. The cron log confirms that the hostname command was launched and completed, but a separate verified reading of `hostname.log` was not captured in the supplied evidence.

### Deletion Detection

The temporary cron configuration was removed.

Wazuh generated an alert with:

- Rule ID: `100632`
- Level: `10`
- File: `/etc/cron.d/lab127_heartbeat`
- Event: `deleted`
- MITRE ATT&CK: `T1053.003`

The subsequent cleanup verification showed that the cron file and `/var/tmp/lab127` directory no longer existed.

## 5. Findings

### Confirmed

- Cronie was installed, and `crond` was active and enabled.
- The inspected cron baseline was captured.
- Wazuh FIM registered the configured cron directories for real-time monitoring.
- Rule `100630` detected creation of the test cron file.
- Rule `100631` detected modification of the test cron file.
- Rule `100632` detected deletion of the test cron file.
- The heartbeat task executed, as demonstrated by timestamp output and cron execution records.
- The test cron file and temporary directory were removed.

### Not Established

- No malicious cron activity was identified through this controlled test.
- No unauthorised user or attacker was attributed to the test configuration.
- The test did not demonstrate compromise, privilege escalation, or persistence by an attacker.
- The investigation did not establish that every possible persistence mechanism on the host was absent.

## 6. Assessment

The lab successfully validated Wazuh detection of cron configuration creation, modification, and deletion. Correlating the FIM alerts with `/var/log/cron` and the heartbeat output provided evidence of scheduled command execution.

In an actual incident, an analyst should inspect the changed file, command, ownership, permissions, modification time, initiating user, related authentication activity, and execution history before determining whether the change is malicious.

**Assessment:** Controlled cron change detection and benign task execution successfully demonstrated.
