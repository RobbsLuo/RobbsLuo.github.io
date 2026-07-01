---
title: An Introduction to PHP-FPM Process Manager Modes
date: 2016-10-10 16:02:10
lang: en
tags:
  - Server
  - PHP
  - Process Manager
categories:
  - Server
description:
  - While a server is running, issues such as PHP-FPM consuming too much memory or spawning too many processes can cause the website or the server itself to become unresponsive. So what does the PHP-FPM process manager actually look like? Let's take a brief look.
---
## Preface
While a server is running, issues such as PHP-FPM consuming too much memory or spawning too many processes can cause the website or the server itself to become unresponsive. So what does the PHP-FPM process manager actually look like? Let's take a brief look.
## Introduction
By inspecting the configuration file, we can see that PHP-FPM has three management modes: `{ static | dynamic | ondemand}`
```bash
vim /etc/php/7.0/fpm/pool.d/www.conf
```
```bash
; Choose how the process manager will control the number of child processes.
; Possible Values:
;   static  - a fixed number (pm.max_children) of child processes;
;   dynamic - the number of child processes are set dynamically based on the
;             following directives. With this process management, there will be
;             always at least 1 children.
;             pm.max_children      - the maximum number of children that can
;                                    be alive at the same time.
;             pm.start_servers     - the number of children created on startup.
;             pm.min_spare_servers - the minimum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is less than this
;                                    number then some children will be created.
;             pm.max_spare_servers - the maximum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is greater than this
;                                    number then some children will be killed.
;  ondemand - no children are created at startup. Children will be forked when
;             new requests will connect. The following parameter are used:
;             pm.max_children           - the maximum number of children that
;                                         can be alive at the same time.
;             pm.process_idle_timeout   - The number of seconds after which
;                                         an idle process will be killed.
; Note: This value is mandatory.
```
## Details
### Static
`pm = static`
Creates and always maintains a fixed number of child processes at startup.
```bash
pm.max_children        # limits the maximum number of php-fpm processes
```
### Dynamic
`pm = dynamic  (default)`
At startup, a fixed number (`start_servers`) of child processes are created. During operation, new child processes may be created, but the total will not exceed `max_children`. The number of child processes fluctuates between `start_servers` and `max_children`, while the number of idle child processes is controlled by `min_spare_servers` and `max_spare_servers`.
```bash
pm.max_children        # limits the maximum number of php-fpm processes
pm.start_servers       # the number of php-fpm processes to start with
pm.min_spare_servers   # the minimum number of idle php-fpm processes
pm.max_spare_servers   # the maximum number of idle php-fpm processes (max_spare_servers <= max_children)

# Official recommended value
start_servers = min_spare_servers + (max_spare_servers - min_spare_servers) / 2
```
### Ondemand
`pm = ondemand`
No child processes are created at startup. Processes are created on demand when requests arrive, and the number will not exceed `max_children`. A process is released once it has been idle for longer than `process_idle_timeout`.
```bash
pm.max_children           # limits the maximum number of php-fpm processes
pm.process_idle_timeout   # the number of seconds an idle process is kept before being released
```

## Recommendations
 - During configuration, you can set `max_children` using the formula `memory / 30M` (where 30M is the maximum memory used by a single process).
 - `static` is suitable for servers with plenty of memory, since creating and releasing child processes consumes server resources (time, memory, CPU...).
 - `dynamic` is suitable for servers with limited memory; it is flexible and saves memory.
 - `ondemand` is not recommended for high-traffic applications, as the overhead of constantly creating and releasing a large number of processes is significant.
 - For production environments, the `static` mode is recommended.
