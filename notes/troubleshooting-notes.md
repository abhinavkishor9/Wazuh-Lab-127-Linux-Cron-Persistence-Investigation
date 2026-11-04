
# Troubleshooting Notes — Linux Cron Persistence Investigation

## 1. Wazuh Manager Restart Timeout

### Issue

The first attempt to restart the Wazuh Manager returned:

`Job for wazuh-manager.service failed because a timeout was exceeded.`

### Actions Taken

A second restart was issued. The subsequent service check returned:

`active`

The Wazuh control status also confirmed:

- `wazuh-syscheckd` was running.
- `wazuh-analysisd` was running.

### Resolution

The subsequent restart succeeded, and the Manager and relevant processes were confirmed active.

The initial timeout should remain documented because the first restart did not complete within the service manager's expected time. The available evidence does not establish the precise cause of that timeout.

## 2. Invalid XML in local_rules.xml

### Issue

The XML validation command reported:

`/var/ossec/etc/rules/local_rules.xml:14: parser error : Extra content at the end of the document`

The error pointed to another top-level group declaration:

`<group name="local,syslog,sshd,">`

This indicated that the file contained multiple top-level XML elements rather than one valid document root.

### Actions Taken

The original file was backed up before repair.

A Python script temporarily wrapped the existing content in a root element, parsed the top-level groups, and consolidated their child elements under one top-level group:

`<group name="local,">`

The repaired file was then parsed again.

### Validation

The following commands completed without reported errors:

- `xmllint --noout /var/ossec/etc/rules/local_rules.xml`
- `/var/ossec/bin/wazuh-analysisd -t`
- `/var/ossec/bin/wazuh-syscheckd -t`

The Wazuh Manager subsequently restarted successfully.

### Important Note

The repair consolidated the contents of the original top-level groups. It did not preserve their individual top-level group names as separate group wrappers. The repaired file should therefore be reviewed if group-specific classification or rule-group behaviour is important to other local rules.

A backup was retained at:

`/var/ossec/etc/rules/local_rules.xml.before-repair-20261104-073448`

A pre-lab backup was also created at:

`/var/ossec/etc/rules/local_rules.xml.pre-lab127`

## 3. FIM Monitoring Confirmation

### Observation

After the successful Manager restart, Wazuh logs showed monitoring entries for:

- `/etc/cron.d`
- `/etc/cron.daily`
- `/etc/cron.hourly`
- `/etc/cron.monthly`
- `/etc/cron.weekly`
- `/var/spool/cron`

The logs also reported that real-time monitoring had started for these directories.

### Result

FIM monitoring was confirmed through the Wazuh Syscheck startup logs. Subsequent creation, modification, and deletion alerts provided functional validation of the configured monitoring.

## 4. Custom-Ticket Integration Errors

### Observation

The Wazuh Manager log contained errors similar to:

`Custom ticket integration failed: <urlopen error [Errno 111] Connection refused>`

The integration process also reported a non-zero exit status.

### Assessment

This indicates that the configured custom-ticket integration could not connect to its target service at the time of execution.

These errors were separate from the cron FIM detection workflow. The supplied evidence does not show that the ticket integration was repaired or that cron alerts successfully created tickets.

## 5. Duplicate Windows Agent Enrollment Warning

### Observation

The Wazuh log reported:

`Duplicate name 'DESKTOP-9MMM37V', rejecting enrollment.`

The message indicated that agent `001` could not be replaced because it was not disconnected.

### Assessment

This was an existing agent enrollment warning and was not part of the Linux cron investigation. No agent replacement or enrollment change was performed as part of this lab.

## 6. Test Cleanup

### Actions

The test cron configuration was removed:

`/etc/cron.d/lab127_heartbeat`

The temporary output files and directory were also targeted for removal.

### Verification

The final checks returned:

- `/etc/cron.d/lab127_heartbeat`: No such file or directory.
- `/var/tmp/lab127`: No such file or directory.

This confirmed removal of the test cron file and temporary directory.

## 7. Troubleshooting Lessons

- Keep one valid top-level XML document element in `local_rules.xml`.
- Back up Wazuh configuration files before making structural changes.
- Validate XML and Wazuh rule configuration before restarting services.
- Treat a service restart timeout separately from the eventual service state.
- Confirm FIM monitoring through both startup logs and actual test events.
- Separate unrelated integration and agent-enrollment errors from the detection result being investigated.
- Verify cleanup explicitly rather than assuming that a removal command succeeded.
