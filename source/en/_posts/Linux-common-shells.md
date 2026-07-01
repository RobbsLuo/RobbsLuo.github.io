---
title: Common Linux Commands
date: 2016-09-23 19:14:35
tags:
  - Linux
  - Shell
categories:
  - Linux
description:
  - A collection of commands I use frequently; this post will be updated over time.
lang: en
---
## Introduction

A collection of commands I use frequently; this post will be updated over time.

## Details

### File Commands
``` bash
$ ls                 ## List the entries (folders and files) under a directory. -a: show hidden files. -l: list view.
$ pwd                ## Print the full path of the current working directory.
$ cd dir             ## Open a directory. - : previous working directory. ~ : current user's home directory.
$ rm                 ## Remove a file. -r: remove a directory. -f: force removal.
$ cp file1 file2     ## Copy. -r: copy a directory, creating it if it does not exist.
$ ln -s file link    ## Create a symbolic link `link` to `file`.
$ touch file         ## Create a file.
$ mkdir dir          ## Create a directory.
$ more file          ## View the contents of `file`.
$ tail -f file       ## View `file` starting from the last 10 lines, following updates.
$ chmod octal file   ## Change the permissions of `file` (e.g. 777, 755).
$ chown              ## Change the owner of a file.
$ Ctrl + W           ## Delete the word under the cursor in the current line.
$ Ctrl + U           ## Delete the entire line.
```

### Process Management
```bash
$ ps            ## Show currently active processes.
$ top           ## Show all running processes.
$ kill pid      ## Kill the process with the given pid.
$ killall proc  ## Kill all processes named `proc`.
$ Ctrl + C      ## Stop the current command.
$ Ctrl + \      ## Stop the current command (use when Ctrl + C does not work).
$ Ctrl + Z      ## Stop the current command; resume it with `fg`.
$ Ctrl + D      ## End the current session (similar to `exit`).
$ !!            ## Repeat the last command.
$ fg            ## Bring a process to the foreground.
$ bg pid        ## Move a process to the foreground.
```

### SSH
```bash
$ ssh user@host         ## Log in to `host` as `user`. -p: port. -i: key file. -vvv: debug mode.
$ ssh-copy-id user@host ## Add your key to `host` to enable passwordless login.
```

### System Information
```bash
$ date                   ## Display the current date and time.
$ ntpdate ntp.ubuntu.com ## Sync the time with the Ubuntu NTP server.
$ cal                    ## Display the calendar for the current month.
$ uptime                 ## Show how long the system has been running since boot.
$ getent passwd          ## List all users.
$ w                      ## Show who is logged in.
$ whoami                 ## Print the current username.
$ finger user            ## Show information about `user`.
$ uname -a               ## Display kernel information.
$ cat /proc/cpuinfo      ## Display CPU information.
$ cat /proc/meminfo      ## Display memory information.
$ man command            ## Display the manual page for `command`.
$ df                     ## Show disk usage.
$ du                     ## Show the disk space used by a directory.
$ free                   ## Show memory and swap usage.
```

### Network
```bash
$ ping host                      ## Ping `host` and print the result.
$ nmap host                      ## Scan `host` for open ports.
$ whois domain                   ## Look up the WHOIS information for `domain`.
$ dig domain                     ## Look up the DNS information for `domain`.
$ dig -x domain                  ## Reverse lookup for `host`.
$ wget file                      ## Download `file`.
$ wget -c file                   ## Resume a broken download.
$ curl -I http://www.q.com       ## Fetch HTTP headers.
```

### Build and Install
```bash
$ ./configure
$ make
$ make install
```

### Search
```bash
$ grep
$ find
```
