---
layout: post
title: Enable re-authorization of a thunderbolt dock
subtitle: Boltd does not re-authorize a dock by default
tags: [centos, fedora, gnome, thunderbold, boltd, boltctl]
author: cmeissner
---

This article describe how to enable `re-auth` for (lenovo) thunderbolt dock.

[Bug 1770579 - Boltd does not re-authorized a Dock station](https://bugzilla.redhat.com/show_bug.cgi?id=1770579)

```shell
$ boltctl domains
 ● domain0 cd010000-0082-8c0e-031c-3a9c06d31105
   ├─ bootacl:  0/16
   └─ security: user
```

```shell
$ boltctl list
 ● Lenovo ThinkPad Thunderbolt 3 Dock
   ├─ type:          peripheral
   ├─ name:          ThinkPad Thunderbolt 3 Dock
   ├─ vendor:        Lenovo
   ├─ uuid:          00c1e830-4529-0801-ffff-ffffffffffff
   ├─ generation:    Thunderbolt 3
   ├─ status:        authorized
   │  ├─ domain:     cd010000-0082-8c0e-031c-3a9c06d31105
   │  ├─ rx speed:   40 Gb/s = 2 lanes * 20 Gb/s
   │  ├─ tx speed:   40 Gb/s = 2 lanes * 20 Gb/s
   │  └─ authflags:  none
   ├─ authorized:    Di 09 Aug 2022 05:58:24 UTC
   ├─ connected:     Di 09 Aug 2022 05:57:42 UTC
   └─ stored:        Do 17 Mär 2022 11:09:54 UTC
      ├─ policy:     iommu
      └─ key:        no
```

```journalctl
$ journalctl -u bolt.service
Mär 09 19:20:41 fedora systemd[1]: Starting Thunderbolt system service...
Mär 09 19:20:41 fedora boltd[1764]: bolt 0.9.1 starting up.
Mär 09 19:20:41 fedora boltd[1764]: manager: initializing store
Mär 09 19:20:41 fedora boltd[1764]: store: located at: /var/lib/boltd
Mär 09 19:20:41 fedora boltd[1764]: store: initializing
Mär 09 19:20:41 fedora boltd[1764]: config: loading user config
Mär 09 19:20:41 fedora boltd[1764]: bouncer: initializing polkit
Mär 09 19:20:41 fedora boltd[1764]: watchdog: enabled [pulse: 90s]
Mär 09 19:20:41 fedora boltd[1764]: udev: initializing udev
Mär 09 19:20:41 fedora boltd[1764]: store: loading domains
Mär 09 19:20:41 fedora boltd[1764]: store: loading devices
Mär 09 19:20:41 fedora boltd[1764]: power: state located at: /run/boltd/power
Mär 09 19:20:41 fedora boltd[1764]: power: force power support: yes
Mär 09 19:20:41 fedora boltd[1764]: udev: found 1 domain
Mär 09 19:20:41 fedora boltd[1764]: udev: enumerating devices
Mär 09 19:20:41 fedora boltd[1764]: probing: adding /sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0 to roots
Mär 09 19:20:41 fedora boltd[1764]: [cd010000-0082-domain0                    ] newly connected [iommu+user] (/sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0/0000:05:00.0/0000:06:00.0/do>
Mär 09 19:20:41 fedora boltd[1764]: security level set to 'user'
Mär 09 19:20:41 fedora boltd[1764]: [cd010000-0082-domain0                    ] domain: registered (bootacl: 16/16)
Mär 09 19:20:41 fedora boltd[1764]: [cd010000-0082-domain0                    ] bootacl: sync start [slots: 16 free: 16]
Mär 09 19:20:41 fedora boltd[1764]: [cd010000-0082-domain0                    ] bootacl: sync done [wrote: no, now free: 16]
Mär 09 19:20:41 fedora boltd[1764]: [cd010000-0082-domain0                    ] udev: uuid is stable: yes (for NHI: 0x15eb)
Mär 09 19:20:41 fedora boltd[1764]: [cd010000-0082-domain0                    ] store: storing newly connected domain
Mär 09 19:20:41 fedora boltd[1764]: journal: opened for 'cd010000-0082'; size: 0 bytes
Mär 09 19:20:41 fedora boltd[1764]: global 'generation' set to '3'
Mär 09 19:20:41 fedora boltd[1764]: [cd010000-0082-ThinkPad P1 Gen3/X1E Gen3 U] device added, status: authorized, at /sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0/0000:05:00.0/0000:06:>
...skipping...
Aug 08 07:58:22 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] connected: connected (/sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0/0000:05:00.0/0000:06:0>
Aug 08 07:58:22 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] auto-auth: authmode: enabled, policy: iommu, iommu: no -> fail
Aug 08 07:58:22 cmeissne.remote.csb boltd[10489]: power: resetting timeout (uevent /sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0/0000:05:00.0/0000:06:00.0/domain0/0-0/0-1)
Aug 08 07:58:22 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] udev: device changed: connected -> connected
Aug 08 07:58:24 cmeissne.remote.csb boltd[10489]: probing: timeout, done: [2000614] (2000000)
Aug 08 07:58:42 cmeissne.remote.csb boltd[10489]: power: setting force_power to OFF
Aug 08 07:58:42 cmeissne.remote.csb boltd[10489]: power: state changed: supported/off
Aug 08 08:02:12 cmeissne.remote.csb boltd[10489]: probing: started [1000]
Aug 08 08:02:12 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] authorize: authorization prepared for 'user' level
Aug 08 08:02:14 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] authorize: finished: ok (status: authorized, flags: 0)
Aug 08 08:02:14 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] udev: device changed: authorized -> authorized
Aug 08 08:02:22 cmeissne.remote.csb boltd[10489]: probing: timeout, done: [2079284] (2000000)
Aug 08 08:03:13 cmeissne.remote.csb boltd[10489]: probing: started [1000]
Aug 08 08:03:16 cmeissne.remote.csb boltd[10489]: probing: timeout, done: [2937723] (2000000)
Aug 08 15:56:38 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] disconnected (/sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0/0000:05:00.0/0000:06:00.0/doma>
Aug 09 07:57:42 cmeissne.remote.csb boltd[10489]: probing: started [1000]
Aug 09 07:57:42 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] parent is cd010000-0082...
Aug 09 07:57:42 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] connected: connected (/sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0/0000:05:00.0/0000:06:0>
Aug 09 07:57:42 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] auto-auth: authmode: enabled, policy: iommu, iommu: no -> fail
Aug 09 07:57:42 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] udev: device changed: connected -> connected
Aug 09 07:57:45 cmeissne.remote.csb boltd[10489]: probing: timeout, done: [2990946] (2000000)
Aug 09 07:58:23 cmeissne.remote.csb boltd[10489]: probing: started [1000]
Aug 09 07:58:23 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] authorize: authorization prepared for 'user' level
Aug 09 07:58:24 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] authorize: finished: ok (status: authorized, flags: 0)
Aug 09 07:58:24 cmeissne.remote.csb boltd[10489]: [00c1e830-4529-ThinkPad Thunderbolt 3 Dock] udev: device changed: authorized -> authorized
Aug 09 07:58:32 cmeissne.remote.csb boltd[10489]: probing: timeout, done: [2027036] (2000000)
```

```shell
$ boltctl forget 00c1e830-4529-0801-ffff-ffffffffffff ; boltctl enroll 00c1e830-4529-0801-ffff-ffffffffffff
 ● Lenovo ThinkPad Thunderbolt 3 Dock
   ├─ type:          peripheral
   ├─ name:          ThinkPad Thunderbolt 3 Dock
   ├─ vendor:        Lenovo
   ├─ uuid:          00c1e830-4529-0801-ffff-ffffffffffff
   ├─ dbus path:     /org/freedesktop/bolt/devices/00c1e830_4529_0801_ffff_ffffffffffff
   ├─ generation:    Thunderbolt 3
   ├─ status:        authorized
   │  ├─ domain:     cd010000-0082-8c0e-031c-3a9c06d31105
   │  ├─ parent:     cd010000-0082-8c0e-031c-3a9c06d31105
   │  ├─ syspath:    /sys/devices/pci0000:00/0000:00:1c.0/0000:04:00.0/0000:05:00.0/0000:06:00.0/domain0/0-0/0-1
   │  ├─ rx speed:   40 Gb/s = 2 lanes * 20 Gb/s
   │  ├─ tx speed:   40 Gb/s = 2 lanes * 20 Gb/s
   │  └─ authflags:  none
   ├─ authorized:    Di 09 Aug 2022 05:58:24 UTC
   ├─ connected:     Di 09 Aug 2022 05:57:42 UTC
   └─ stored:        Do 17 Mär 2022 11:09:54 UTC
      ├─ policy:     auto
      └─ key:        no
```
