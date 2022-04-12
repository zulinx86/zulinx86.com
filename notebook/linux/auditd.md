---
layout: post
title: auditd
---

## What is auditd?
- auditd is the userspace component to the Linux Auditing System.
- It's responsible for writing audit records to the disk.



## Configuration files
- `/etc/audit/auditd.conf`: configuration file for audit daemon (auditd)
- `/etc/audit/audit.rules`: audit rules to be loaded at startup
- `/etc/audit/rules.d/*.rules`: rules files to be compiled into `/etc/audit/audit.rules` by `augenrules` command.



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
- `augenrules` is a script that merges all comopnent audit rules files found in the audit rules directory `/etc/audit/rules.d/` and places the merged file in `/etc/audit/audit.rules`
- `auditrules --check`: test if rules have changed and need updating without overwriting `/etc/audit/audit.rules`.
- `auditrules --load`: load old or newly built rules into the kernel.


### [auditctl](https://man7.org/linux/man-pages/man8/auditctl.8.html)
- `auditctl` is a utility for configuring the audit system or loading rules. During startup of auditd, the rules in `/etc/audit/audit.rules` are read by `auditctl` and loaded into the kernel.
- Configuration options
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
- Examples
	- To see all syscalls made by a specific process:
		- `auditctl -a always,exit -S all -F pid=1005`
	- To see files opened by a specific user:
		- `auditctl -a always,exit -S openat -F audit=510`
	- To watch a file for changes (2 ways):
		- `auditctl -w /etc/shadow -p wa`
		- `auditctl -a always,exit -F path=/etc/shadow -F perm=wa`
	- To recursively watch a directory for change (2 ways):
		- `auditctl -w /etc/ -p wa`
		- `auditctl -a always,exit -F path=/etc/ -F perm=wa`


### aureport
- `aureport` produces summary reports of the audit system logs.
- Options
	- `--auth`: report about authentication attempts.
	- `--comm`: report about commands run.
	- `--config`: report about config changes.
	- `--event`: report about events.
	- `--file`: report about files and af_unix sockets.
	- `--login`: report about logins.
	- `--pid`: report about processes.
	- `--syscall`: report about syscalls.
	- `--user`: report about users.


### ausearch
- `ausearch` can search the audit log file for certain events using various keys or other characteristics.
- Options
	- `-c, --comm comm-name`: search for an event based on the given command name.
	- `-p, --pid process-id`: search for an event matching the given process ID.
	- `-sc, --syscall syscall`: search for an event matching given syscall.
	- `-ts, --start [start-date] [start-time]`: search for events with timestamps equal to or after the given start time.
	- `-te, --end [end-date] [end-time]`: search for events with timestamps equal to or before the given end time.
	- `-ui, --uid user-id`: search for an event with the given user ID.
	- `-ul, --loginuid login-id`: search for an event with the given login user ID.
	- `-x, --executable executalbe`: search for an event matching the given executable name.


### autrace
- `autrace` traces individual processes in a fasion similar to strace.



## Hands-On
### Default Setting
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
$ sudo auditctl -w /home/ec2-user/ -p rwxa
$ sudo auditctl -l
-w /home/ec2-user/ -p rwxa

$ touch hello.txt
$ echo hello > hello.txt
$ cat hello.txt
hello
$ rm hello.txt

$ sudo auditctl -D
No rules

$ sudo cat /var/log/audit/audit.log | grep hello
type=PATH msg=audit(1649768935.858:125025): item=1 name="hello.txt" inode=13253038 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1649768940.030:125026): item=1 name="hello.txt" inode=13253038 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1649768942.710:125027): item=0 name="hello.txt" inode=13253038 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1649768945.306:125030): item=1 name="hello.txt" inode=13253038 dev=ca:01 mode=0100664 ouid=1000 ogid=1000 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0

$ sudo aureport --file

File Report
===============================================
# date time file syscall success exe auid event
===============================================
(snipped)
17187. 04/12/22 13:08:55 /home/ec2-user 257 yes /usr/bin/touch 1000 125025
17188. 04/12/22 13:09:00 /home/ec2-user 257 yes /usr/bin/bash 1000 125026
17189. 04/12/22 13:09:02 hello.txt 257 yes /usr/bin/cat 1000 125027
17190. 04/12/22 13:09:05 /home/ec2-user 263 yes /usr/bin/rm 1000 125030
```
