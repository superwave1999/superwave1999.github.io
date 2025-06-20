---
title: "Ubuntu 24.04 Debloat"
date: 2025-06-22
tags:
  - ubuntu
  - debloat
  - server
  - homelab
categories: [documentation]
---

## The objective

After our neat Ubuntu install, I want to remove some components from our system that **I** believe to be corporate, bloat and corporate bloat.

## Basic Rundown

- We don't want ANY automatic background processes
- My use-case is docker-centric, so no snapd
- No Canonical ads

## Remove packages
```bash
apt purge --auto-remove snapd
apt purge --auto-remove unattended-upgrades
apt purge --auto-remove software-properties-common
```

If not running in cloud:
```bash
apt purge --auto-remove cloud-init
```

## Remove ads on login

```bash
apt purge landscape-common ubuntu-advantage-tools update-notifier-common

chmod -x /etc/update-motd.d/50-motd-news
chmod -x /etc/update-motd.d/91-release-upgrade
chmod -x /etc/update-motd.d/91-contact-ua-esm-status
```

```bash
systemctl stop landscape-client.timer
systemctl disable landscape-client.timer
systemctl stop motd-news.timer
systemctl disable motd-news.timer
```

> As usual, if I find any minor issues or any further notable improvements, I'll update the guide.