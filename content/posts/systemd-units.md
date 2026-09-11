---
title: "Writing your first systemd unit"
date: 2026-09-10
categories: ["linux-unix"]
tags: ["systemd", "administration", "linux"]
---
A systemd service unit describes how to run and supervise a daemon.

```ini
[Unit]
Description=My app
After=network.target

[Service]
ExecStart=/usr/bin/myapp
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start it with `systemctl enable --now myapp`.
