# II. SAN network

## 1. Storage machines

> **Toutes les manips de cettes section sont à réaliser sur `sto1` et `sto2`.**

### A. Disks and RAID

➜ **Ajouter des disques aux machines `sto1` et `sto2`**

🌞 **Configurer des RAID**

```bash
[root@sto1-1 ~]# cat /proc/mdstat
Personalities : [raid1] 
md2 : active raid1 sdg[1] sdb[0]
      1046528 blocks super 1.2 [2/2] [UU]
      
md1 : active raid1 sde[1] sdf[0]
      1046528 blocks super 1.2 [2/2] [UU]
      
md0 : active raid1 sdc[1] sdd[0]
      1046528 blocks super 1.2 [2/2] [UU]

[root@sto2-1 ~]# cat /proc/mdstat
Personalities : [raid1] 
md2 : active raid1 sdg[1] sdf[0]
      1046528 blocks super 1.2 [2/2] [UU]
      
md1 : active raid1 sde[1] sdd[0]
      1046528 blocks super 1.2 [2/2] [UU]
      
md0 : active raid1 sdc[1] sdb[0]
      1046528 blocks super 1.2 [2/2] [UU]
```

🌞 **Prouvez que vous avez 3 volumes RAID prêts à l'emploi**

```bash
[root@sto2-1 ~]# lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk  
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part  
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0    1G  0 disk  
└─md0              9:0    0 1022M  0 raid1 /mnt/raid1
sdc                8:32   0    1G  0 disk  
└─md0              9:0    0 1022M  0 raid1 /mnt/raid1
sdd                8:48   0    1G  0 disk  
└─md1              9:1    0 1022M  0 raid1 /mnt/raid2
sde                8:64   0    1G  0 disk  
└─md1              9:1    0 1022M  0 raid1 /mnt/raid2
sdf                8:80   0    1G  0 disk  
└─md2              9:2    0 1022M  0 raid1 /mnt/raid3
sdg                8:96   0    1G  0 disk  
└─md2              9:2    0 1022M  0 raid1 /mnt/raid3
```

### B. iSCSI target

🌞 **Installer `target`**

```bash
[root@sto1-1 ~]# dnf install targetcli -y
[root@sto2-1 ~]# dnf install targetcli -y

```

🌞 **Démarrer le service `target`**

```bash
[root@sto2-1 ~]# systemctl status target
○ target.service - Restore LIO kernel target configuration
     Loaded: loaded (/usr/lib/systemd/system/target.service; enabled; preset: disabl>
     Active: inactive (dead)
[root@sto1-1 ~]# systemctl status target
○ target.service - Restore LIO kernel target configuration
     Loaded: loaded (/usr/lib/systemd/system/target.service; enabled; preset:>
     Active: inactive (dead)
```

🌞 **Configurer les *targets iSCSI***

```bash
Note: block backstore preferred for best results
Created fileio data-chunk1 with size 1071644672
/> /backstores/fileio create name=data-chunk2 file_or_dev=/dev/md1
Note: block backstore preferred for best results
Created fileio data-chunk2 with size 1071644672
/> /backstores/fileio create name=data-chunk3 file_or_dev=/dev/md2/chunk3
Attempting to create file for new fileio backstore, need a size
/> /backstores/fileio create name=data-chunk3 file_or_dev=/dev/md2
Note: block backstore preferred for best results
Created fileio data-chunk3 with size 1071644672
/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 3]
  | | o- data-chunk1 ................................................................... [/dev/md0 (1022.0MiB) write-back activated]
  | | | o- alua ................................................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | | o- data-chunk2 ................................................................... [/dev/md1 (1022.0MiB) write-back activated]
  | | | o- alua ................................................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | | o- data-chunk3 ................................................................... [/dev/md2 (1022.0MiB) write-back activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 3]
  | o- iqn.2024-12.tp2.b3:data-chunk1 .................................................................................... [TPGs: 1]
  | | o- tpg1 ............................................................................................... [no-gen-acls, no-auth]
  | |   o- acls .......................................................................................................... [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator ...................................................... [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 .......................................................................... [lun0 fileio/data-chunk1 (rw)]
  | |   o- luns .......................................................................................................... [LUNs: 1]
  | |   | o- lun0 ............................................................... [fileio/data-chunk1 (/dev/md0) (default_tg_pt_gp)]
  | |   o- portals .................................................................................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ..................................................................................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk2 .................................................................................... [TPGs: 1]
  | | o- tpg1 ............................................................................................... [no-gen-acls, no-auth]
  | |   o- acls .......................................................................................................... [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:data-chunk2:chunk1-initiator ...................................................... [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 .......................................................................... [lun0 fileio/data-chunk2 (rw)]
  | |   o- luns .......................................................................................................... [LUNs: 1]
  | |   | o- lun0 ............................................................... [fileio/data-chunk2 (/dev/md1) (default_tg_pt_gp)]
  | |   o- portals .................................................................................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ..................................................................................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk3 .................................................................................... [TPGs: 1]
  |   o- tpg1 ............................................................................................... [no-gen-acls, no-auth]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2024-12.tp2.b3:data-chunk3:chunk1-initiator ...................................................... [Mapped LUNs: 1]
  |     |   o- mapped_lun0 .......................................................................... [lun0 fileio/data-chunk3 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ............................................................... [fileio/data-chunk3 (/dev/md2) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
/> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/target/backup/.
Configuration saved to /etc/target/saveconfig.json

/> /iscsi create iqn.2024-12.tp2.b3:data-chunk1
Created target iqn.2024-12.tp2.b3:data-chunk1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/> /iscsi create iqn.2024-12.tp2.b3:data-chunk2
Created target iqn.2024-12.tp2.b3:data-chunk2.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/> /iscsi create iqn.2024-12.tp2.b3:data-chunk3
Created target iqn.2024-12.tp2.b3:data-chunk3.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
```

## 2. Chunks machine

### A. Simple iSCSI

🌞 **Installer les tools iSCSI sur `chunk1.tp2.b3`**

```bash
[root@sto2-1 ~]# apt install iscsi-initiator-utils -y
-bash: apt: command not found
[root@sto2-1 ~]# dnf install iscsi-initiator-utils -y
Last metadata expiration check: 0:53:13 ago on Thu Dec  5 14:24:01 2024.
Dependencies resolved.
=====================================================================================
 Package                          Arch     Version                    Repo      Size
=====================================================================================
Installing:
 iscsi-initiator-utils            x86_64   6.2.1.9-1.gita65a472.el9   baseos   383 k
Installing dependencies:
 iscsi-initiator-utils-iscsiuio   x86_64   6.2.1.9-1.gita65a472.el9   baseos    80 k
 isns-utils-libs                  x86_64   0.101-4.el9                baseos   100 k
```

🌞 **Configurer un iSCSI initiator**

```bash
[root@chunk3 ~]# sudo iscsiadm -m discoverydb -t st -p 10.3.1.1:3260 --discover
sudo iscsiadm -m discoverydb -t st -p 10.3.1.2:3260 --discover
sudo iscsiadm -m discoverydb -t st -p 10.3.2.1:3260 --discover
sudo iscsiadm -m discoverydb -t st -p 10.3.2.2:3260 --discover
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.2.1:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.2.1:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.2.1:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.2.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.2.2:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.2.2:3260,1 iqn.2024-12.tp2.b3:data-chunk3
[root@chunk1 ~]# cat /etc/iscsi/initiatorname.iscsi 
InitiatorName=iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator

sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.1.1 --login
sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.1.2 --login
sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.2.1 --login
sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.2.2 --login
```

🌞 **Modifier la configuration du démon iSCSI**

```bash
[root@chunk1 ~]# cat /etc/iscsi/initiatorname.iscsi 
InitiatorName=iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
```

🌞 **Prouvez que la configuration est prête**

```bash

[root@chunk1 ~]# lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                8:0    0   20G  0 disk 
├─sda1             8:1    0    1G  0 part /boot
└─sda2             8:2    0   19G  0 part 
  ├─rl_vbox-root 253:0    0   17G  0 lvm  /
  └─rl_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                8:16   0 1022M  0 disk 
sdc                8:32   0 1022M  0 disk 
sdd                8:48   0 1022M  0 disk 
sde                8:64   0 1022M  0 disk 
sr0               11:0    1 1024M  0 rom  
[root@chunk1 ~]# iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.1.9
Target: iqn.2024-12.tp2.b3:data-chunk1 (non-flash)
	Current Portal: 10.3.2.1:3260,1
	Persistent Portal: 10.3.2.1:3260,1
        [...]
	Current Portal: 10.3.1.1:3260,1
	Persistent Portal: 10.3.1.1:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
		Iface IPaddress: 10.3.1.101
		Iface HWaddress: default
		Iface Netdev: default
		[...]
	Current Portal: 10.3.1.2:3260,1
	Persistent Portal: 10.3.1.2:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
		Iface IPaddress: 10.3.1.101
		Iface HWaddress: default
		Iface Netdev: default
		SID: 7
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
        [...]
	Current Portal: 10.3.2.2:3260,1
	Persistent Portal: 10.3.2.2:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
		Iface IPaddress: 10.3.2.101
		Iface HWaddress: default
		Iface Netdev: default
		SID: 8
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 120
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
```

🌞 **Installer les outils multipath sur `chunk1.tp2.b3`**

```bash
[root@chunk1 ~]# dnf install device-mapper-multipath -y
Rocky Linux   12 kB/s | 4.1 kB     00:00     Rocky Linux   13 kB/s | 4.5 kB     00:00     Rocky Linux   14 MB/s | 8.3 MB     00:00     Rocky Linux  8.1 kB/s | 2.9 kB     00:00     Dependencies resolved.
=============================================
[...]
Upgraded:
  device-mapper-9:1.02.198-2.el9.x86_64      
  device-mapper-event-9:1.02.198-2.el9.x86_64
  device-mapper-event-libs-9:1.02.198-2.el9.x86_64
  device-mapper-libs-9:1.02.198-2.el9.x86_64 
  kpartx-0.8.7-32.el9.x86_64                 
  lvm2-9:2.03.24-2.el9.x86_64                
  lvm2-libs-9:2.03.24-2.el9.x86_64           
Installed:
  device-mapper-multipath-0.8.7-32.el9.x86_64
  device-mapper-multipath-libs-0.8.7-32.el9.x86_64


```

🌞 **Configurer le fichier `/etc/multipath.conf`**

```bash
[root@chunk1 etc]# cat /etc/multipath.conf | head -n 10
blacklist_exceptions {
	device {
		vendor	"IBM"
		product	"S/390.*"
	}
}

defaults {
  user_friendly_names yes
  find_multipaths yes

```

🌞 **Démarrer le service `multipathd`**

```bash
[root@chunk1 etc]# systemctl status multipathd
● multipathd.service - Device-Mapper Multipath Device Controller
     Loaded: loaded (/usr/lib/systemd/system/multipathd.service; enabled; preset: enabled)
     Active: active (running) since Fri 2024-12-06 09:21:21 CET; 2s ago
TriggeredBy: ○ multipathd.socket
    Process: 2053 ExecStartPre=/sbin/modprobe -a scsi_dh_alua scsi_dh_emc scsi_dh_rdac dm-multipath (code=exited, status=0/SUCCESS)
    Process: 2057 ExecStartPre=/sbin/multipath -A (code=exited, status=0/SUCCESS)
   Main PID: 2058 (multipathd)
     Status: "up"
      Tasks: 7
     Memory: 19.5M
        CPU: 90ms
     CGroup: /system.slice/multipathd.service
             └─2058 /sbin/multipathd -d -s
```

🌞 **Et euh c'est tout, il est smart enough**

```bash
[root@chunk1 etc]# lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk  
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part  
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0 1022M  0 disk  
└─mpatha         253:2    0 1022M  0 mpath 
sdc                8:32   0 1022M  0 disk  
└─mpatha         253:2    0 1022M  0 mpath 
sdd                8:48   0 1022M  0 disk  
└─mpathb         253:3    0 1022M  0 mpath 
sde                8:64   0 1022M  0 disk  
└─mpathb         253:3    0 1022M  0 mpath 
sr0               11:0    1 1024M  0 rom   
[root@chunk1 etc]# multipath -ll
1602.459062 | /etc/multipath.conf line 26, invalid keyword in the multipath section: path_checker
mpatha (36001405b4749195c5464576a491525dd) dm-2 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405a828c5747cc546eda35801396) dm-3 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
```

## 3. Formatage et montage

🌞 **Créez une partition sur les devices `mpatha` et `mpathb`**

```bash
[root@chunk1 etc]# fdisk /dev/mapper/mpatha 

Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xd036d41a.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (16384-2093055, default 16384): w
Value out of range.
First sector (16384-2093055, default 16384): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (16384-2093055, default 2093055): 

Created a new partition 1 of type 'Linux' and of size 1014 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Re-reading the partition table failed.: Invalid argument

The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or partx(8).
```

🌞 **Formatez en `xfs` les partitions**
```bash
[root@chunk1 ~]# lsblk -f
NAME             FSTYPE       FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
[...]
sdb              mpath_member                                                                      
└─mpatha                                                                                           
  └─mpatha1      xfs                         13d2266c-66c5-4fc0-bae3-893a5c48c25d                  
sdc              mpath_member                                                                      
└─mpatha                                                                                           
  └─mpatha1      xfs                         13d2266c-66c5-4fc0-bae3-893a5c48c25d                  
sdd              mpath_member                                                                      
└─mpathb                                                                                           
  └─mpathb1      xfs                         497683c6-4058-49c9-b324-e353538467fe                  
sde              mpath_member                                                                      
└─mpathb                                                                                           
  └─mpathb1      xfs                         497683c6-4058-49c9-b324-e353538467fe   
```

🌞 **Point de montage `/mnt/data_chunk1`**

```bash
[root@chunk2 ~]# cat /etc/systemd/system/mnt-data_chunk1.mount 
[Unit]
Description=Mount for /mnt/data_chunk1
After=network-online.target
Wants=network-online.target

[Mount]
What=/dev/mapper/mpatha
Where=/mnt/data_chunk1
Type=xfs
Options=defaults

[Install]
WantedBy=multi-user.target
[root@chunk2 ~]# lsblk -f
NAME             FSTYPE       FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
[...]
sdd              mpath_member                                                                      
└─mpatha                                                                                           
  └─mpatha1      xfs                         8fcba48c-4af9-4078-89a7-a0b9df411301    911.1M     4% /mnt/data_chunk1
sde              mpath_member                                                                      
└─mpatha                                                                                           
  └─mpatha1      xfs                         8fcba48c-4af9-4078-89a7-a0b9df411301    911.1M     4% /mnt/data_chunk1
sr0
[root@chunk2 ~]# df -h
Filesystem                Size  Used Avail Use% Mounted on
/dev/mapper/mpatha1       950M   39M  912M   5% /mnt/data_chunk1
```

🌞 **Point de montage `/mnt/data_chunk2`**

```bash
[root@chunk2 ~]# cat /etc/systemd/system/mnt-data_chunk2.mount 
[Unit]
Description=Mount for /mnt/data_chunk2
After=network-online.target
Wants=network-online.target

[Mount]
What=/dev/mapper/mpathb
Where=/mnt/data_chunk2
Type=xfs
Options=defaults

[Install]
WantedBy=multi-user.target
[root@chunk2 ~]# lsblk -f
NAME             FSTYPE       FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
[...]
sdb              mpath_member                                                                      
└─mpathb                                                                                           
  └─mpathb1      xfs                         f6674e0b-9fd3-4c34-a751-91ee610d4152    911.1M     4% /mnt/data_chunk2
sdc              mpath_member                                                                      
└─mpathb                                                                                           
  └─mpathb1      xfs                         f6674e0b-9fd3-4c34-a751-91ee610d4152    911.1M     4% /mnt/data_chunk2
sr0
[root@chunk2 ~]# df -h
Filesystem                Size  Used Avail Use% Mounted on
/dev/mapper/mpathb1       950M   39M  912M   5% /mnt/data_chunk2

```

## 4. Tests

### A. Simulation de panne

🌞 **Simuler une coupure réseau**

```bash
[root@chunk2 ~]# date
Fri Dec  6 10:23:25 CET 2024

mpatha (3600140557c66dd047214999a651eb037) dm-2 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=0 status=active
| `- 5:0:0:0 sdd 8:48 failed faulty running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
mpathb (3600140532c8eec0828442df8df844bb1) dm-3 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=0 status=enabled
| `- 7:0:0:0 sdb 8:16 failed faulty running
`-+- policy='service-time 0' prio=50 status=active
  `- 8:0:0:0 sdc 8:32 active ready running

[root@chunk2 ~]# date
Fri Dec  6 10:24:43 CET 2024

mpatha (3600140557c66dd047214999a651eb037) dm-2 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 failed ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
mpathb (3600140532c8eec0828442df8df844bb1) dm-3 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 7:0:0:0 sdb 8:16 failed ready running
`-+- policy='service-time 0' prio=50 status=active
  `- 8:0:0:0 sdc 8:32 active ready running

```

### B. Jouer avec les paramètres

🌞 **Resimuler une panne**

```bash
[root@chunk2 ~]# date
Fri Dec  6 10:50:3 CET 2024

mpatha (3600140557c66dd047214999a651eb037) dm-2 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=0 status=active
| `- 5:0:0:0 sdd 8:48 failed faulty running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
mpathb (3600140532c8eec0828442df8df844bb1) dm-3 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=0 status=enabled
| `- 7:0:0:0 sdb 8:16 failed faulty running
`-+- policy='service-time 0' prio=50 status=active
  `- 8:0:0:0 sdc 8:32 active ready running

[root@chunk2 ~]# date
Fri Dec  6 10:50:27 CET 2024

mpatha (3600140557c66dd047214999a651eb037) dm-2 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 failed ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
mpathb (3600140532c8eec0828442df8df844bb1) dm-3 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 7:0:0:0 sdb 8:16 failed ready running
`-+- policy='service-time 0' prio=50 status=active
  `- 8:0:0:0 sdc 8:32 active ready running
```

## 5. Replicate

🌞 **Preuve du setup**

```bash
[root@chunk2 ~]# lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk  
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part  
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0 1022M  0 disk  
└─mpathb         253:3    0 1022M  0 mpath 
  └─mpathb1      253:5    0 1014M  0 part  /mnt/data_chunk2
sdc                8:32   0 1022M  0 disk  
└─mpathb         253:3    0 1022M  0 mpath 
  └─mpathb1      253:5    0 1014M  0 part  /mnt/data_chunk2
sdd                8:48   0 1022M  0 disk  
└─mpatha         253:2    0 1022M  0 mpath 
  └─mpatha1      253:4    0 1014M  0 part  /mnt/data_chunk1
sde                8:64   0 1022M  0 disk  
└─mpatha         253:2    0 1022M  0 mpath 
  └─mpatha1      253:4    0 1014M  0 part  /mnt/data_chunk1
sr0               11:0    1 1024M  0 rom   
[root@chunk2 ~]# df -h
Filesystem                Size  Used Avail Use% Mounted on
devtmpfs                  4.0M     0  4.0M   0% /dev
tmpfs                     882M     0  882M   0% /dev/shm
tmpfs                     353M  5.0M  348M   2% /run
/dev/mapper/rl_vbox-root   17G  1.2G   16G   8% /
/dev/sda1                 960M  223M  738M  24% /boot
tmpfs                     177M     0  177M   0% /run/user/0
/dev/mapper/mpatha1       950M   39M  912M   5% /mnt/data_chunk1
/dev/mapper/mpathb1       950M   42M  909M   5% /mnt/data_chunk2
[root@chunk2 ~]# iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.1.9
Target: iqn.2024-12.tp2.b3:data-chunk2 (non-flash)
	Current Portal: 10.3.2.2:3260,1
	Persistent Portal: 10.3.2.2:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk1-initiator
		Iface IPaddress: 10.3.2.102
		Iface HWaddress: default
		Iface Netdev: default
		SID: 10
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 6	State: running
		scsi6 Channel 00 Id 0 Lun: 0
			Attached scsi disk sde		State: running
	Current Portal: 10.3.1.1:3260,1
	Persistent Portal: 10.3.1.1:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk1-initiator
		Iface IPaddress: 10.3.1.102
		Iface HWaddress: default
		Iface Netdev: default
		SID: 5
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 7	State: running
		scsi7 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdb		State: running
	Current Portal: 10.3.2.1:3260,1
	Persistent Portal: 10.3.2.1:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk1-initiator
		Iface IPaddress: 10.3.2.102
		Iface HWaddress: default
		Iface Netdev: default
		SID: 6
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 8	State: running
		scsi8 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdc		State: running
	Current Portal: 10.3.1.2:3260,1
	Persistent Portal: 10.3.1.2:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk1-initiator
		Iface IPaddress: 10.3.1.102
		Iface HWaddress: default
		Iface Netdev: default
		SID: 9
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 5	State: running
		scsi5 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdd		State: running
[root@chunk2 ~]# multipath -ll
6996.373607 | /etc/multipath.conf line 26, invalid keyword in the multipath section: path_checker
mpatha (3600140557c66dd047214999a651eb037) dm-2 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
mpathb (3600140532c8eec0828442df8df844bb1) dm-3 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 7:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=active
  `- 8:0:0:0 sdc 8:32 active ready running

[root@chunk3 ~]# lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk  
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part  
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0 1022M  0 disk  
└─mpathb         253:3    0 1022M  0 mpath 
  └─mpathb1      253:5    0 1014M  0 part  
sdc                8:32   0 1022M  0 disk  
└─mpathb         253:3    0 1022M  0 mpath 
  └─mpathb1      253:5    0 1014M  0 part  
sdd                8:48   0 1022M  0 disk  
└─mpatha         253:2    0 1022M  0 mpath 
  └─mpatha1      253:4    0 1014M  0 part  
sde                8:64   0 1022M  0 disk  
└─mpatha         253:2    0 1022M  0 mpath 
  └─mpatha1      253:4    0 1014M  0 part  
sr0               11:0    1 1024M  0 rom   
[root@chunk3 ~]# df -h
Filesystem                Size  Used Avail Use% Mounted on
devtmpfs                  4.0M     0  4.0M   0% /dev
tmpfs                     882M     0  882M   0% /dev/shm
tmpfs                     353M  5.0M  348M   2% /run
/dev/mapper/rl_vbox-root   17G  1.2G   16G   8% /
/dev/sda1                 960M  223M  738M  24% /boot
tmpfs                     177M     0  177M   0% /run/user/0
[root@chunk3 ~]# multipath -ll
7073.694832 | /etc/multipath.conf line 26, invalid keyword in the multipath section: path_checker
mpatha (360014054dab0c38acbe481b985650efa) dm-2 LIO-ORG,data-chunk3
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 13:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 14:0:0:0 sde 8:64 active ready running
mpathb (3600140568639b32dcf64b85992c43877) dm-3 LIO-ORG,data-chunk3
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0  sdc 8:32 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0  sdb 8:16 active ready running
[root@chunk3 ~]# iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.1.9
Target: iqn.2024-12.tp2.b3:data-chunk3 (non-flash)
	Current Portal: 10.3.1.2:3260,1
	Persistent Portal: 10.3.1.2:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk1-initiator
		Iface IPaddress: 10.3.1.103
		Iface HWaddress: default
		Iface Netdev: default
		SID: 11
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 13	State: running
		scsi13 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdd		State: running
	Current Portal: 10.3.2.2:3260,1
	Persistent Portal: 10.3.2.2:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk1-initiator
		Iface IPaddress: 10.3.2.103
		Iface HWaddress: default
		Iface Netdev: default
		SID: 12
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 14	State: running
		scsi14 Channel 00 Id 0 Lun: 0
			Attached scsi disk sde		State: running
	Current Portal: 10.3.1.1:3260,1
	Persistent Portal: 10.3.1.1:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk1-initiator
		Iface IPaddress: 10.3.1.103
		Iface HWaddress: default
		Iface Netdev: default
		SID: 3
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 5	State: running
		scsi5 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdc		State: running
	Current Portal: 10.3.2.1:3260,1
	Persistent Portal: 10.3.2.1:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk1-initiator
		Iface IPaddress: 10.3.2.103
		Iface HWaddress: default
		Iface Netdev: default
		SID: 4
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 5
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 6	State: running
		scsi6 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdb		State: running

```
