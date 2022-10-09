# DFIR - Linux artefacts overview

### Filesystem timelining

###### Filesystem types supported timestamps

| Filesystem | atime (access) | mtime (modification) | ctime (metadata change) | crtime (creation / birth) | Comment |
|------------|----------------|----------------------|-------------------------|---------------------------|---------|
| `ext2` <br> `ext3` | x | x | x | - | |
| `ext4` | x | x | x | x | |
| `XFS` | x | x | x | x* | * Since `XFS v5` |

###### Timelining

```bash
find <DIRECTORY> -xdev -print0 | xargs -0 stat -c 'crtime="%w" crtime_epoch="%W" mtime="%y" mtime_epoch="%Y" ctime="%z" ctime_epoch="%Z" atime="%x" atime_epoch="%X" size_bytes="%s" userID="%u" username="%U" groupID="%g" groupname="%G" access="%a" access_pretty="%A" filetype="%F" filename="%n" filename_deref="%N"'
```

### Artefacts

| Name | Information relative to | Location |
|------|-------------------------|----------|
| `alternatives` logs | System information. <br><br> Logs of the `update-alternatives` utility, used to manage *alternatives* (i.e symbolic links to a given command). | `/var/log/alternatives.log` |
| `Apache` webserver logs | Web access <br><br> Logs of the `Apache` webserver. | Debian / Ubuntu: <br> ` /var/log/apache2/access.log` <br> `/var/log/apache2/error.log` <br><br> RHEL / Red Hat / CentOS / Fedora : <br> `/var/log/httpd/access_log` <br> `/var/log/httpd/error_log` <br><br> FreeBSD: <br> `/var/log/httpd-access.log` <br> `/var/log/httpd-error.log` <br><br> Custom definition for access (`CustomLog ` section) or error (`ErrorLog` section) logs: <br> `/etc/httpd/conf/httpd.conf` <br> `/etc/apache2/apache2.conf` <br> `/usr/local/etc/apache22/httpd.conf` |
| `apt` / `apt-get` logs | Software installation. <br><br> Logs of `apt-get` / `apt` operations, including packets installation. | Current log file: <br> `/var/log/apt/history.log` <br><br> Rotated log archives: <br>`/var/log/apt/history.log.*.gz` |
| `aptitude` logs | Software installation. <br><br> Logs of the `aptitude` utility (front-end to `apt`) operations, including packets installation. | `/var/log/aptitude` |
| `at` jobs (`atd` daemon) | Persistence. <br><br> Scheduled jobs that are configured using the `at` command-line utility to be run exactly one time. By default, any user can create `at` jobs. The jobs are executed as shell (bash, zsh, etc.) scripts. <br><br> Each `at` jobs is represented by a file, which contains metadata information as comments, the environment variables for the execution, and the configured shell script / commands. The filename follows a specific format (`[a=]<JOB_NUMBER_5_CHAR><TIMESAMP_8_CHAR>`) and gives additional information about the job. The following information can be deduced from the file name and the file itself: <br> - File created or last modified timestamp => when the `at` job was created. <br> - Username, `uid`, and `gid` of the user that created the job as shell comments in the file. <br> - Filename first char: `a` => job is pending or `=` => job is running. <br> - Filename 5 next chars => job id. <br> - Filename 8 next (and last) chars => hex-encoded minutes since `epoch` timestamp. Can be converted to retrieve the `epoch` timestamp of execution by converting to decimal and multiplying by 60. | Configured `at` jobs locations, each files representing a single `at` job: <br> `/var/spool/at/` <br> `/var/spool/cron/atjobs/` <br><br> Configuration files that define the users that can or cannot create `at` jobs: <br> `/etc/at.allow` <br> `/etc/at.deny` <br><br> Output of currently running `at` jobs, saved as email text files: <br> `/var/spool/at/spool/` <br> `/var/spool/cron/atspool/` <br><br> Number of `at` jobs that have been created (already executed, executing, or scheduled): <br> `/var/spool/at/.SEQ` <br> `/var/spool/cron/atjobs/.SEQ` <br><br> Trace of previous `at` jobs execution can be found in: <br> - Session opening by the `atd` daemon events in `syslog` or `journal` logs. <br> - `at` jobs email sent events in local email logs. |
| Audit `auditd` framework (`audit` logs) | Non default, can be configured to log multiple types of operations, such as authentication successes or failures, process executions, file accesses, user commands executed in a TTY, etc. <br><br> Each record / log entry contain a `msg` field, composed of a timestamp and an unique ID. Multiple records generated as part of the same Auditd event can share the same `msg` field. For example, `cat /etc/passwd` can generate `SYSCALL` + `EXECVE` records for the execution of `cat` and a `PATH` record for the access to the `/etc/passwd` file. <br><br> The `type` field contains the type of the record: <br><br> - User authentication and access: `USER_LOGIN_SUCCESS`, `USER_LOGIN_FAILED`, `USER_AUTH_SUCCESS`, `USER_AUTH_FAILED`, `USER_START_SUCCESS`, `USER_START_FAILED`, `SESSION_TERMINATED`. <br><br> - Process execution: `EXECVE` and `SYSCALL`. <br><br> - Filesystem access: `PATH` (for relative or absolute file access), `CWD` (current working directory, useful to reconstruct full path if a relative path has been recorded in `PATH` records) and `OPENAT`. <br><br> - Commands entered in a `TTY` console: `TTY` or by users: `USER_CMD`. <br><br> - Full command-line of process: `PROCTITLE`. The associated `proctitle` field MAY be encoded in hexadecimal. <br><br> - Network socket connections: `SOCKADDR`. The associated `saddr` field contains IP and port information, and can be interpreted directly at event generation (if `log_format = ENRICHED` is set), or with `ausearch -i` or [simple scripting](https://gist.github.com/Qazeer/3aaa6be263380483d68159cae6f33fd2). <br><br> - Account activity: `ADD_USER` or `ADD_GROUP`. <br><br> - More record types are listed in the [RedHat documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security_guide/sec-audit_record_types). <br><br> If present, the `auid` field defines the ID of the user upon login and remains the same even if the user's identity changes (for instance with `su`). <br><br> If present, `uid` / `gid` and `euid` / `egid` fields define the user / group IDs and the effective user / group IDs of the audited process. <br><br> If present, the `tty` and `ses` fields define respectively the terminal and session from which the audited process was invoked. <br><br> For `SYSCALL` records, the `aX` field(s) define the arguments / parameters of the syscall, represented by unsigned long long integers and as such cannot be used to determine the values taken by the arguments. | Configuration file notably defining the path of the log files: <br> `/etc/auditd.conf` <br><br> Configuration defining the rules to apply: <br> `/etc/audit/audit.rules` <br> Rules best practice: https://github.com/Neo23x0/auditd <br><br> Current log files (default location): <br> `/var/log/audit/audit.log` <br> `/var/log/audit/audit.log.1` <br><br> Rotated log archives (default location): <br> `/var/log/audit/audit.log.*.gz` <br><br> The `aureport` and `ausearch` utilities can (and if possible should) be used to search the `auditd` log files. <br><br> Example: <br> `aureport -i [--login \| --executable \| ...] [--summary] -if <AUDIT_LOG_FILE>` |
| `Authorization (auth)` logs | Authentication information and `sudo` commands. More precisely, usage of authorization systems: successful or unsuccessful logins, sudo commands, etc. <br><br> Usually generated by the `AUTH` and `AUTHPRIV` facilities of the `syslog` daemon. `AUTH` regroups the authentication events / messages while `AUTHPRIV` regroups the elevation of privileges events / messages (such as commands executed through `sudo`). <br><br> Notably includes: <br><br> - Successful or unsuccessful logins to the `sshd` deamon. The authentication types (password, pubkey, etc.) or reason of failure (unknown user, invalid password) is specified. <br><br> - Commands executed with elevated privileges using `sudo`. | *Location of the `auth` logs depend of the `syslog` daemon configuration (refer to the "Syslog daemon configuration" artefact below for more information).* <br><br> [Debian / Ubuntu based systems] <br><br> Default location: <br> `/var/log/auth.log` <br> `/var/log/auth.log.1` <br><br> Rotated log archives: <br> `/var/log/auth.log.*.gz` <br><br> [RedHat / CentOS based systems] <br><br> Default location (for `AUTHPRIV` logs): <br> `/var/log/secure` |
| `dpkg` logs | Software installation. <br><br> Logs of `dpkg` operations, including packets installed / removed through the utility. | Current log files: <br> `/var/log/dpkg.log` <br> `/var/log/dpkg.log.1` <br><br> Rotated log archives: <br>`/var/log/dpkg.log.*.gz` |
| Environment variables information | System information. <br><br> Contains system-wide or user scoped persistent environment variables. | System-wide configuration file: <br> `/etc/environment` <br><br> Initialization scripts can also be used to define system-wide or user scoped environment variables. |
| Hostname information | System information. <br><br> Contains the hostname of the system. | `/etc/hostname` |
| Mounted filesystems information | System information. <br><br> Contains information on the mounted file systems, such as partition types (ext3 / ext4, etc.). | Configuration: <br> `/etc/fstab` <br><br> Mount logs (such as `Mounting` operation / keyword): <br> `/var/log/dmesg` |
| Shell initialization scripts | Persistence / system information. <br><br> System-wide or user scoped scripts that are executed by all shells during their initialization. | User scoped initialization script: <br> `<USER_HOME_DIR>/.profile` <br><br> System-wide initialization scripts: <br> `/etc/profile` <br> `/etc/profile.d/*` <br> `/etc/skel/.profile` (Not used if `<USER_HOME_DIR>/.bash_profile` or `<USER_HOME_DIR>/.bash_login` exist). |
| `SSH` authorization keys | Persistence / system information. <br><br> Specifies the `SSH` keys that can be used for logging into the user account for which the file is configured, thus allowing permanent access as that user. | Configuration of the `SSH` authorization keys: <br> `/etc/ssh/sshd_config` `AuthorizedKeysFile`directive. <br><br> Default `SSH` authorization keys location: <br> `<USER_HOME_DIR>/.ssh/authorized_keys` <br> `<USER_HOME_DIR>/.ssh/authorized_keys2`
| `SSH` known hosts | Lateral movement: possible `SSH` outgoing connections. <br><br> System-wide or user scoped known `SSH` keys for remote hosts. Usually collected, and user-validated, from the remote hosts when connecting for the first time. <br><br> The remote hosts hostname and IP address can be either stored in clear-text or hashed if `HashKnownHosts` is set to "yes" in the `SSH` client `ssh_config` configuration file. <br> Even if the hosts information are hashed, the following command can be used to check whether the specified hostname is present in the given known hosts file: <br> `ssh-keygen -l -f <KNOWN_HOST_FILE> -F <HOSTNAME>`. <br> Additionally, `John` can be used to bruteforce known hosts files: <br> `john --format=known_hosts <KNOWN_HOST_FILE>` <br><br> `nmap -sL -Pn -n 192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 \| grep '^Nmap scan report for' \| cut -d ' ' -f 5 > IP_list.txt` <br> `john --wordlist=IP_list.txt --format=known_hosts <KNOWN_HOST_FILE>` <br><br> One of the few endpoint disk artefacts to identify outgoing `SSH` connections. | System-wide known hosts: <br> `/etc/ssh/ssh_known_hosts` <br><br> `<USER_HOME_DIR>/.ssh/known_hosts` |
| `Syslog` daemon configuration | System information. <br><br> The `Syslog` deamon configuration file(s) notably define where the messages / events received by the `Syslog` daemon will be outputted. The messages are usually written as plaintext files under `/var/log/` but can also be sent over the network. <br><br> Example of a configuration file writing logs to common files: <br><br> auth,authpriv.* /var/log/auth.log <br> \*.\*;auth,authpriv.none -/var/log/syslog <br> kern.* -/var/log/kern.log <br> mail.* -/var/log/mail.log | `/etc/syslog.conf` <br> `/etc/rsyslog.conf` <br> `/etc/rsyslog.d/*.conf` <br> `/etc/syslog­ng.conf` <br> `/etc/syslog­ng/*` |
| Timezone information | System information. <br><br> Contains the timezone of the system. | `/etc/timezone` <br><br> `/etc/adjtime` <br><br> `/etc/localtime` |
| Webshell | Command execution / Persistence. <br><br> Simply put, webshells are script files that are executed by a webserver. Webshells are notably leveraged to: <br><br> - execute code / commands on the underlying operating system following the exploitation of a web vulnerability (unrestricted file upload, remote code execution, etc.) <br><br> - maintain persistence following the compromise of an host exposing a webserver (usually Internet facing). | Usual locations: <br><br> `/var/www/html` <br> `/usr/local/www/` <br><br> `/etc/nginx` <br><br> `/etc/apache2` <br><br> `/srv/` <br> `/srv/www` <br><br> `...` <br><br> Webshells can be uncovered by: <br> - Yara rules aimed at webshells such as [`Neo23x0's gen_webshells.yar`](https://github.com/Neo23x0/signature-base/blob/master/yara/gen_webshells.yar) or [`thor-webshells.yar`](https://github.com/nsacyber/WALKOFF-Apps/blob/master/AlienVault/signature-base/yara/thor-webshells.yar): <br> `yara -r <YARA_RULE_PATH> <WEBSERVER_ROOT>` <br><br> - Reviewing added files or modifications in legitimate files using code repository or a fresh install of the application if possible. <br><br> - Manually by looking for known webshell patterns (`Runtime.getRuntime().exec`, `eval`, `system`, etc.), obfuscated script files, or files modified during the targeted timeframe. <br><br> - Reviewing the webserver access logs if available, looking for exploitation IoCs, unusual requests, large response size, etc. |
| Login records `*tmp` | Authentication information. <br><br> `utmp` / `utmpx`: currently logged users. <br> `wtmp` / `wtmpx`: all current and past logins, with additional details on system reboots, etc. <br> `btmp` / `btmpx`: all bad login attempts. <br><br> `*tmp` login records are not stored in clear-text and must be parsed with adequate utilities, such as `utmpdump <*TMP_FILE>`. <br><br> The `*tmpx` files are extended database files that supersede the `*tmp` files on some distributions. | Linux: <br> `/var/run/utmp` <br> `/var/log/wtmp` <br> `/var/log/btmp` <br><br> Solaris: <br> `/var/adm/utmp` (deprecated) <br> `/var/adm/utmpx` <br> `/var/adm/wtmp` (deprecated) <br> `/var/adm/wtmpx` <br><br> FreeBSD 9.0: <br> `/var/run/utx.active` (`utmp` equivalent) <br> `/var/log/utx.log` (`wtmp` equivalent) |

TODO

/etc/security/lastlog	Specifies the path to the lastlog file.

/etc/group	Contains the basic attributes of groups.

/etc/security/group	Contains the extended attributes of groups.

/etc/passwd	Contains the basic attributes of users.

/etc/security/passwd	Contains password information.

/etc/security/environ	Contains the environment attributes of users.

/etc/security/user	Contains the extended attributes of users.

/etc/security/limits	Contains the process resource limits of users.
https://www.ibm.com/docs/en/aix/7.1?topic=formats-lastlog-file-format

--------------------------------------------------------------------------------

### References

https://wiki.debian-fr.xyz/Consulter_les_logs_:_quoi,_o%C3%B9_et_comment_chercher_%3F

https://nostarch.com/download/samples/PracticalLinuxForensics_Ch5_072721.pdf

https://blog.codeasite.com/how-do-i-find-apache-http-server-log-files/

https://sematext.com/blog/auditd-logs-auditbeat-elasticsearch-logsene/

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-understanding_audit_log_files

https://www.elastic.co/fr/blog/grokking-the-linux-authorization-logs

https://unix.stackexchange.com/questions/31549/is-it-possible-to-find-out-the-hosts-in-the-known-hosts-file

https://pberba.github.io/security/2021/11/22/linux-threat-hunting-for-persistence-sysmon-auditd-webshell/

https://en.wikipedia.org/wiki/Utmp