---
layout: post
title:  "Systemd service that is always restarted"
tags: [autologin,console,getty,ubuntu,systemd]
---
Just using `Restart` and `RestartSec` is not enough: systemd services have start rate limiting enabled by default. If service is started more than `StartLimitBurst` times in `StartLimitIntervalSec` seconds is it not permitted to start any more. This parameters are inherited from `DefaultStartLimitIntervalSec`(default 10s) and `DefaultStartLimitBurst`(default 5) in `systemd-system.conf`.

Use `systemctl edit foobar.service` or manually edit `/etc/systemd/system/foobar.service.d/override.conf` and run `systemctl deamon-reload`.

```ini
[Service]
Restart=always
# time to sleep before restarting a service
RestartSec=1

[Unit]
# StartLimitIntervalSec in recent systemd versions
StartLimitInterval=0
```

In recent systemd versions `StartLimitInterval` was renamed to `StartLimitIntervalSec`.

Links:

* [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
* [systemd.unit(5)](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)
* [systemd-system.conf(5)](https://www.freedesktop.org/software/systemd/man/systemd-system.conf.html)
