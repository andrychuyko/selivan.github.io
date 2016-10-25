---
layout: post
title:  "Fix 502 error on php-fpm service reload"
tags: php
---

In default configuration php-fpm produces errors 502(bad gateway) on reloading. This is really annoying, if you need to do it often. For example, if you use tool like [deployer](https://deployer.org/) to deploy your application into new directory and then switch symlink to it.

Problem is in php-fpm parameter `process_control_timeout`([documentation](http://php.net/manual/en/install.fpm.configuration.php)). It controls time for child process to process signls from master and defaults to 0. So effectively reload kills worker, and you get error 502. I recommend setting this parameter to same value as `max_execution_time`, so worker have time to finish processing request.

One more nuance about php-fpm: unlike other Unix services, it is reloaded on SIGUSR2, not SIGHUP. In Ubuntu, old upstart versions could not have custom signals for actions. In latter version in was fixed, but service description file `/etc/init/php5-fpm.conf` or `/etc/init/php7.0-fpm.conf` is still broken. For Ubuntu 12.04 and 14.04 you should add line `reload signal USR2` to the file, otherwise `reload php5-fpm` will effectively restart it. In later Ubuntu versions systemd is used, so this problem is irrelevant.
