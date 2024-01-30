---
title: Raspberry Pi 5 gnome-core 설치
description:
date: 2024-01-01
summary: "Raspberry Pi 5 Lite (64-bit) + gnome-core"
---

## Environment

- Raspberry Pi OS Lite (64-bit)
  - release : 2023-12-11

## Shell

```shell
sudo apt update

sudo apt upgrade -y

sudo apt install gnome-core -y

sudo systemctl set-default graphical.target

sudo reboot
```
