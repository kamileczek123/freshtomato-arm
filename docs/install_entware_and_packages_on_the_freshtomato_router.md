# Install Entware and Packages on the FreshTomato Router

## Install Entware Base

Entware will be intalled into `/opt` path on enabled USB Storage.

See the [Entware Wiki](https://github.com/Entware/Entware/wiki).

1. Check router architecture and kernel version

```bash
uname -a

# output
Linux router1.lo 2.6.36.4brcmarm #10 SMP PREEMPT Thu Jul 17 23:47:32 CEST 2025 armv7l Tomato
```

Kernel version: `2.6.36.4brcmarm`
Architecture version: `armv7l`

2. Install Entware

Use sources according to discovered router architecture and kernel version from [bin.entware.net](https://bin.entware.net/), in this case `armv7sf` + `k2.6`. Run installation script while on router via terminal

```bash
wget -O - http://bin.entware.net/armv7sf-k2.6/installer/generic.sh | sh
```

## Install Entware Packages

1. Install basic packages

```bash
opkg install parted rsync tar openssh-sftp-server grep lsblk mc atop htop curl
```

2. Install python3 to enable Ansible further

```bash
opkg install python3
```
