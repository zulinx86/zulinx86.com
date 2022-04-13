---
layout: post
title: Audit
---

## What is Audit?
- The Linux Audit system provides a way to track security-relevant information on your system.
- Audit system architecture
	- Audit system consists of two main parts:
		- the user-space applications and utilities
		- the kernel-side system call processing
	- The kernel component receives system calls from user-space applications and filters them.



## Files
- `/etc/audit/auditd.conf`: configuration file for audit daemon (auditd)
	- `write_logs`: whether or not to write logs to the disk.
	- `log_file`: the log file path.
	- `log_group`: the group that is applied to the log file's permissions.
	- `log_format`: how the information should be stored on disk.
	- `log_events`: whether or not to include local events.
	- `max_log_file`: the maximum size in megabytes.
	- `max_log_file_action`: what action is taken once the max file size limit has been reached.
	- `space_left`: if the free space in the filesystem containing `log_file` drops below this value, the auditd takes the action specified by `space_left_action`.
	- `space_left_action`: what action is taken when the left space get below `space_left`.
	- `admin_space_left`: if the free space in the filesystem containg `log_file` drops below this value, the auditd takes the action specified by `amdin_space_left_action`.
	- `admin_space_left_action`: what action is taken when the left space get below `amdin_space_left`. 
	- `disk_full_action`: what action is taken when the disk is full.
	- `disk_error_action`: what action is taken when there is an error detected when writing audit events to disk or rotating logs.
	- `num_logs`: the number of log files to keep if rotate.
- `/etc/audit/audit.rules`: audit rules to be loaded at startup
- `/etc/audit/rules.d/*.rules`: rules files to be compiled into `/etc/audit/audit.rules` by `augenrules` command.
- `/var/log/audit/audit.log`: log file



## Commands
```
$ rpm -ql audit | grep bin
/sbin/audispd
/sbin/auditctl
/sbin/auditd
/sbin/augenrules
/sbin/aureport
/sbin/ausearch
/sbin/autrace
/usr/bin/aulast
/usr/bin/aulastlog
/usr/bin/ausyscall
/usr/bin/auvirt
```


### [augenrules](https://man7.org/linux/man-pages/man8/augenrules.8.html)
- `augenrules` is a script that merges all audit rules files found in the audit rules directory `/etc/audit/rules.d/` and places the merged file in `/etc/audit/audit.rules`
- `auditrules --check`: test if rules have changed and need updating without overwriting `/etc/audit/audit.rules`.
- `auditrules --load`: load old or newly built rules into the kernel.


### [auditctl](https://man7.org/linux/man-pages/man8/auditctl.8.html)
- `auditctl` is a utility for configuring the audit system or loading rules. During startup of auditd, the rules in `/etc/audit/audit.rules` are read by `auditctl` and loaded into the kernel.
- Control options
	- `-D`: delete all rules.
	- `-b backlog`: set max number of audit buffers allowed.
	- `-f [0..2]`: set failure mode (`0`=silent, `1`=printk, `2`=panic).
	- `-e [0..2]`: set enabled flag. `0` disables auditing temporarily.
- Status options
	- `-l`: list all rules 1 per line.
	- `-s`: report the kernel's audit subsystem status.
- Rule options
	- `-a [list,action|action,list]`: append rule to the end of `list` with `action`.
		- `list`
			- `exit`: add a rule to the syscall exit list. used upon exit from a syscall to determine if an audit event should be created.
			- `exclude`: add a rule to the event type exclusion filter list. used to filter events that you do not want to see.
		- `action`
			- `never`: no audit records will be generated.
			- `always`: allocate an audit context, always fill it in at syscall entry time, and always write out a record at syscall exit time.
	- `-A list,action`: add rule to the beginning list with action.
	- `-d list,action`: delete rule from list with action.
	- `-F [n=v | n<v | n>v | n<=v | n>=v | n&v | n&=v]`: add condition for rule (formated as `<name><operation><field>`).
		- `pid`: process ID.
		- `ppid`: parent's process ID.
		- `exe`: absolute path to application.
		- `path`: full path of file to watch.
		- `dir`: full path of directory to watch.
		- `perm`: permission filter for file operations. See `-p`.
		- `uid`: user ID.
		- `auid`: the original ID the user logged in with. refered to as loginuid.
	- `-p [r|w|x|a]`: describe the permission access type that a file system watch will trigger on. (`r`=read, `w`=write, `x`=execute, `a`=attribute change)
	- `-S [syscall name or number|all]`: specify system call.
	- `-w path`: insert a watch for the file system object at `path`.
	- `-W path`: remove a watch for the file system object at `path`.
- Defining Rules
	- File system rules: `auditctl -w <path-to-file-or-dir> -p <permissions>`
	- System call rules: `auditctl -a <list>,<action> -S <syscall> -F <field>=<value>`



### [aureport](https://man7.org/linux/man-pages/man8/aureport.8.html)
- `aureport` produces summary reports of the audit system logs.
- Options
	- `-i, --interpret`: interpret numeric entities into text.
	- `-au, --auth`: report about authentication attempts.
	- `--comm`: report about commands run.
	- `-c, --config`: report about config changes.
	- `-e, --event`: report about events.
	- `-f, --file`: report about files and af_unix sockets.
	- `-l, --login`: report about logins.
	- `-k, --key`: report about audit rule keys.
	- `-p, --pid`: report about processes.
	- `-s, --syscall`: report about syscalls.
	- `-u, --user`: report about users.
	- `-x, --executable`: report about executables.


### [ausearch](https://man7.org/linux/man-pages/man8/ausearch.8.html)
- `ausearch` can search the audit log file for certain events using various keys or other characteristics.
- Options
	- `-i, --interpret`: interpret numeric entities into text.
	- `-c, --comm comm-name`: search for an event based on the given command name.
	- `-p, --pid process-id`: search for an event matching the given process ID.
	- `-sc, --syscall syscall`: search for an event matching given syscall.
	- `-ts, --start [start-date] [start-time]`: search for events with timestamps equal to or after the given start time.
	- `-te, --end [end-date] [end-time]`: search for events with timestamps equal to or before the given end time.
	- `-ui, --uid user-id`: search for an event with the given user ID.
	- `-ul, --loginuid login-id`: search for an event with the given login user ID.
	- `-x, --executable executalbe`: search for an event matching the given executable name.


### [autrace](https://man7.org/linux/man-pages/man8/autrace.8.html)
- `autrace` traces individual processes in a fasion similar to strace.



## Understanding Audit Log Files
### Sample 1
```
type=SYSCALL msg=audit(1364481363.243:24287): arch=c000003e syscall=2 success=no exit=-13 a0=7fffd19c5592 a1=0 a2=7fffd19c4b50 a3=a items=1 ppid=2686 pid=3538 auid=1000 uid=1000 gid=1000 euid=1000 suid=1000 fsuid=1000 egid=1000 sgid=1000 fsgid=1000 tty=pts0 ses=1 comm="cat" exe="/bin/cat" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="sshd_config"
```
- `type=SYSCALL`: syscall record type.
- `msg=audit(1364481363.243:24287):`: time stamp and unique ID.
- `arch=c000003e`: CPU arch encoded in hexadecimal notation. (`c000003e` means x86_64)
- `syscall=2`: syscall number. (`/usr/include/asm/unistd_64.h` is available to check the corresponding syscall)
- `success=no`: whether the syscall succeeded or failed.
- `exit=-13`: exit code returned by the syscall.
- `a0=7fffd19c5592 a1=0 a2=7fffd19c4b50 a3=a`: first four arguments of the syscall encoded in hexadecimal notation.
- `comm="cat"`: command-line name.
- `exe="/bin/cat"`: path to executable file


### Sample 2
```
type=CWD msg=audit(1364481363.243:24287):  cwd="/home/shadowman"
```
- `type=CWD`: current working directory (CWD) record type.
- `msg=audit(1364481363.243:24287)`: time stamp and unique ID.
- `cwd="/home/shadowman"`: path to the directory.


### Sample 3
```
type=PATH msg=audit(1364481363.243:24287): item=0 name="/etc/ssh/sshd_config" inode=409248 dev=fd:00 mode=0100600 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:etc_t:s0  objtype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0
```
- `type=PATH`: path record type.
- `msg=audit(1364481363.243:24287):`: time stamp and unique ID
- `name="/etc/ssh/sshd_config"`: path to file or directory that was passed to the syscall.
- `inode=409248`: inode number associated with the file or directory. `find / -inum <inode> -print` returns path.
- `dev=fd:00`: major and minor ID of the device.
- `mode=0100600`: file or directory permissions.



## Hands-On
### Default Setting
```sh
$ sudo cat /etc/audit/auditd.conf
#
# This file controls the configuration of the audit daemon
#

local_events = yes
write_logs = yes
log_file = /var/log/audit/audit.log
log_group = root
log_format = RAW
flush = INCREMENTAL_ASYNC
freq = 50
max_log_file = 8
num_logs = 5
priority_boost = 4
disp_qos = lossy
dispatcher = /sbin/audispd
name_format = NONE
##name = mydomain
max_log_file_action = ROTATE
space_left = 75
space_left_action = SYSLOG
verify_email = yes
action_mail_acct = root
admin_space_left = 50
admin_space_left_action = SUSPEND
disk_full_action = SUSPEND
disk_error_action = SUSPEND
use_libwrap = yes
##tcp_listen_port = 60
tcp_listen_queue = 5
tcp_max_per_addr = 1
##tcp_client_ports = 1024-65535
tcp_client_max_idle = 0
enable_krb5 = no
krb5_principal = auditd
##krb5_key_file = /etc/audit/audit.key
distribute_network = no
```

The following shows that `/etc/audit/audit.rules` file is generated by `/etc/audit/rules.d/*.rules` files.
```sh
$ sudo cat /etc/audit/audit.rules
## This file is automatically generated from /etc/audit/rules.d
-D
-b 8192
-f 1

$ sudo ls /etc/audit/rules.d/
audit.rules

$ sudo cat /etc/audit/rules.d/audit.rules 
## First rule - delete all
-D

## Increase the buffers to survive stress events.
## Make this bigger for busy systems
-b 8192

## Set failure mode to syslog
-f 1
```

The following shows that there are no rules and auditd is set up as described in `/etc/audit/audit.rules`.
```sh
$ sudo auditctl -l
No rules

$ sudo auditctl -s
enabled 1
failure 1
pid 2542
rate_limit 0
backlog_limit 8192
lost 0
backlog 0
backlog_wait_time 15000
loginuid_immutable 0 unlocked
```


### Watch directory
```sh
$ sudo auditctl -w /home/ec2-user/ -p rwxa -k my_rule
$ sudo auditctl -l
-w /home/ec2-user -p rwxa -k my_rule

$ touch hello.txt
$ echo hello > hello.txt
$ cat hello.txt
hello
$ rm hello.txt

$ sudo auditctl -D
No rules

$ sudo cat /var/log/audit/audit.log | grep hello
type=PATH msg=audit(1649816776.126:125885): item=1 name="hello.txt" inode=12591969 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1649816780.946:125886): item=1 name="hello.txt" inode=12591969 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1649816783.578:125887): item=0 name="hello.txt" inode=12591969 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1649816787.178:125890): item=1 name="hello.txt" inode=12591969 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0

$ sudo ausearch -k my_rule -i
----
type=CONFIG_CHANGE msg=audit(04/13/22 02:26:02.149:125874) : auid=ec2-user ses=104 op=add_rule key=my_rule list=exit res=yes 
----
type=PROCTITLE msg=audit(04/13/22 02:26:16.126:125885) : proctitle=touch hello.txt 
type=PATH msg=audit(04/13/22 02:26:16.126:125885) : item=1 name=hello.txt inode=12591969 dev=ca:01 mode=file,664 ouid=ec2-user ogid=ec2-user rdev=00:00 nametype=CREATE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0 
type=PATH msg=audit(04/13/22 02:26:16.126:125885) : item=0 name=/home/ec2-user inode=13261880 dev=ca:01 mode=dir,700 ouid=ec2-user ogid=ec2-user rdev=00:00 nametype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0 
type=CWD msg=audit(04/13/22 02:26:16.126:125885) : cwd=/home/ec2-user 
type=SYSCALL msg=audit(04/13/22 02:26:16.126:125885) : arch=x86_64 syscall=openat success=yes exit=3 a0=0xffffffffffffff9c a1=0x7ffc8b70b6fa a2=O_WRONLY|O_CREAT|O_NOCTTY|O_NONBLOCK a3=0x1b6 items=2 ppid=1062 pid=1093 auid=ec2-user uid=ec2-user gid=ec2-user euid=ec2-user suid=ec2-user fsuid=ec2-user egid=ec2-user sgid=ec2-user fsgid=ec2-user tty=pts0 ses=104 comm=touch exe=/usr/bin/touch key=my_rule 
----
type=PROCTITLE msg=audit(04/13/22 02:26:20.946:125886) : proctitle=-bash 
type=PATH msg=audit(04/13/22 02:26:20.946:125886) : item=1 name=hello.txt inode=12591969 dev=ca:01 mode=file,664 ouid=ec2-user ogid=ec2-user rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0 
type=PATH msg=audit(04/13/22 02:26:20.946:125886) : item=0 name=/home/ec2-user inode=13261880 dev=ca:01 mode=dir,700 ouid=ec2-user ogid=ec2-user rdev=00:00 nametype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0 
type=CWD msg=audit(04/13/22 02:26:20.946:125886) : cwd=/home/ec2-user 
type=SYSCALL msg=audit(04/13/22 02:26:20.946:125886) : arch=x86_64 syscall=openat success=yes exit=3 a0=0xffffffffffffff9c a1=0x198f980 a2=O_WRONLY|O_CREAT|O_TRUNC a3=0x1b6 items=2 ppid=1061 pid=1062 auid=ec2-user uid=ec2-user gid=ec2-user euid=ec2-user suid=ec2-user fsuid=ec2-user egid=ec2-user sgid=ec2-user fsgid=ec2-user tty=pts0 ses=104 comm=bash exe=/usr/bin/bash key=my_rule 
----
type=PROCTITLE msg=audit(04/13/22 02:26:23.578:125887) : proctitle=cat hello.txt 
type=PATH msg=audit(04/13/22 02:26:23.578:125887) : item=0 name=hello.txt inode=12591969 dev=ca:01 mode=file,664 ouid=ec2-user ogid=ec2-user rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0 
type=CWD msg=audit(04/13/22 02:26:23.578:125887) : cwd=/home/ec2-user 
type=SYSCALL msg=audit(04/13/22 02:26:23.578:125887) : arch=x86_64 syscall=openat success=yes exit=3 a0=0xffffffffffffff9c a1=0x7ffcfedce6fe a2=O_RDONLY a3=0x0 items=1 ppid=1062 pid=1094 auid=ec2-user uid=ec2-user gid=ec2-user euid=ec2-user suid=ec2-user fsuid=ec2-user egid=ec2-user sgid=ec2-user fsgid=ec2-user tty=pts0 ses=104 comm=cat exe=/usr/bin/cat key=my_rule 
----
type=PROCTITLE msg=audit(04/13/22 02:26:27.178:125890) : proctitle=rm hello.txt 
type=PATH msg=audit(04/13/22 02:26:27.178:125890) : item=1 name=hello.txt inode=12591969 dev=ca:01 mode=file,664 ouid=ec2-user ogid=ec2-user rdev=00:00 nametype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0 
type=PATH msg=audit(04/13/22 02:26:27.178:125890) : item=0 name=/home/ec2-user inode=13261880 dev=ca:01 mode=dir,700 ouid=ec2-user ogid=ec2-user rdev=00:00 nametype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0 
type=CWD msg=audit(04/13/22 02:26:27.178:125890) : cwd=/home/ec2-user 
type=SYSCALL msg=audit(04/13/22 02:26:27.178:125890) : arch=x86_64 syscall=unlinkat success=yes exit=0 a0=0xffffff9c a1=0x12ce290 a2=0x0 a3=0x166 items=2 ppid=1062 pid=1095 auid=ec2-user uid=ec2-user gid=ec2-user euid=ec2-user suid=ec2-user fsuid=ec2-user egid=ec2-user sgid=ec2-user fsgid=ec2-user tty=pts0 ses=104 comm=rm exe=/usr/bin/rm key=my_rule 
----
type=PROCTITLE msg=audit(04/13/22 02:26:30.190:125895) : proctitle=auditctl -D 
type=SOCKADDR msg=audit(04/13/22 02:26:30.190:125895) : saddr={ fam=netlink nlnk-fam=16 nlnk-pid=0 } 
type=SYSCALL msg=audit(04/13/22 02:26:30.190:125895) : arch=x86_64 syscall=sendto success=yes exit=1080 a0=0x3 a1=0x7ffe5432b3b0 a2=0x438 a3=0x0 items=0 ppid=1096 pid=1097 auid=ec2-user uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts0 ses=104 comm=auditctl exe=/usr/sbin/auditctl key=(null) 
type=CONFIG_CHANGE msg=audit(04/13/22 02:26:30.190:125895) : auid=ec2-user ses=104 op=remove_rule key=my_rule list=exit res=yes 
```



## Links
- [Chapter 7. System Auditing Red Hat Enterprise Linux 7 - Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-system_auditing)
- [RHEL Audit System Reference - Red Hat Customer Portal](https://access.redhat.com/articles/4409591)
- [Home · linux-audit/audit-documentation Wiki](https://github.com/linux-audit/audit-documentation/wiki)