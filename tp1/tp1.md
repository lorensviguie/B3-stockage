# TP1 : Single-machine storage

## Sommaire

- [TP1 : Single-machine storage](#tp1--single-machine-storage)
  - [Sommaire](#sommaire)
- [I. Fundamentals](#i-fundamentals)
- [II. Partitioning](#ii-partitioning)
- [III. RAID](#iii-raid)
  - [1. Simple RAID](#1-simple-raid)
  - [2. Break it](#2-break-it)
  - [3. Spare disk](#3-spare-disk)
  - [4. Grow](#4-grow)
- [IV. NFS](#iv-nfs)


# I. Fundamentals

🌞 **Lister tous les périphériques de stockage branchés à la VM**

```bash
[root@storage ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   10G  0 disk 
├─sda1        8:1    0    1G  0 part /boot
└─sda2        8:2    0    9G  0 part 
  ├─rl-root 253:0    0    8G  0 lvm  /
  └─rl-swap 253:1    0    1G  0 lvm  [SWAP]
sdb           8:16   0    5G  0 disk 
sdc           8:32   0    5G  0 disk 
sdd           8:48   0    5G  0 disk 
sde           8:64   0    5G  0 disk 
sdf           8:80   0    5G  0 disk 
sdg           8:96   0    5G  0 disk 
sr0          11:0    1  1.7G  0 rom  
```

🌞 **Lister toutes les partitions des périphériques de stockage**

voir soleil au dessus

🌞 **Effectuer un test SMART sur le disque**

```bash
[root@storage ~]# smartctl
-bash: smartctl: command not found
```

🌞 **Espace disque...**

```bash
[root@storage ~]# df -h | grep mapper
/dev/mapper/rl-root  8.0G  1.5G  6.5G  19% /
```

🌞 **Inodes**

```bash
[root@storage ~]# df -i
Filesystem           Inodes IUsed   IFree IUse% Mounted on
devtmpfs              93396   434   92962    1% /dev
tmpfs                 98417     1   98416    1% /dev/shm
tmpfs                819200   631  818569    1% /run
/dev/mapper/rl-root 4192256 35685 4156571    1% /
/dev/sda1            524288   366  523922    1% /boot
tmpfs                 19683    15   19668    1% /run/user/0
```

🌞 **Latence disque**

```bash
[root@storage ~]# ioping /dev/sda
4 KiB <<< /dev/sda (block device 10 GiB): request=1 time=226.3 us (warmup)
4 KiB <<< /dev/sda (block device 10 GiB): request=2 time=204.7 us
4 KiB <<< /dev/sda (block device 10 GiB): request=3 time=277.4 us
4 KiB <<< /dev/sda (block device 10 GiB): request=4 time=286.4 us
^C
--- /dev/sda (block device 10 GiB) ioping statistics ---
3 requests completed in 768.5 us, 12 KiB read, 3.90 k iops, 15.2 MiB/s
generated 4 requests in 3.66 s, 16 KiB, 1 iops, 4.37 KiB/s
min/avg/max/mdev = 204.7 us / 256.2 us / 286.4 us / 36.6 us
```

🌞 **Déterminer la taille du cache *filesystem***

```bash
[root@storage ~]# grep -i "cache" /proc/meminfo
Cached:           224688 kB
SwapCached:        11412 kB
```

# II. Partitioning

Ici on utilise un des disques supplémentaires branchés à la VM : `sdb`.

> Assurez-vous que LVM est installé sur votre OS avant de continuer.

🌞 **Ajouter `sdb` comme Physical Volume LVM**

```bash
[root@storage ~]# pvs
  PV         VG Fmt  Attr PSize  PFree
  /dev/sda2  rl lvm2 a--  <9.00g    0 
[root@storage ~]# pvcreate sdb
  No device found for sdb.
[root@storage ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[root@storage ~]# pvs
  PV         VG Fmt  Attr PSize  PFree
  /dev/sda2  rl lvm2 a--  <9.00g    0 
  /dev/sdb      lvm2 ---   5.00g 5.00g
  ```

🌞 **Créer un Volume Group LVM nommé `storage`**

```bash
[root@storage ~]# vgcreate storage /dev/sdb
  Volume group "storage" successfully created
[root@storage ~]# vgs
  VG      #PV #LV #SN Attr   VSize  VFree 
  rl        1   2   0 wz--n- <9.00g     0 
  storage   1   0   0 wz--n- <5.00g <5.00g
```

🌞 **Créer un Logical Volume LVM**

```bash
[root@storage ~]# lvcreate -L 2G -n smol_data storage
  Logical volume "smol_data" created.
[root@storage ~]# lvs
  LV        VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root      rl      -wi-ao---- <8.00g                                                    
  swap      rl      -wi-ao----  1.00g                                                    
  smol_data storage -wi-a-----  2.00g 
```

🌞 **Créer un deuxième Logical Volume LVM**

```bash
[root@storage ~]# lvcreate -L 2G -n big_data storage
  Logical volume "big_data" created.
[root@storage ~]# lvs
  LV        VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root      rl      -wi-ao---- <8.00g                                                    
  swap      rl      -wi-ao----  1.00g                                                    
  big_data  storage -wi-a-----  2.00g                                                    
  smol_data storage -wi-a-----  2.00g  
```

🌞 **Créez un système de fichiers sur les deux LVs**

```bash
[root@storage ~]# sudo mkfs.ext4 /dev/mapper/storage-smol_data
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done                            
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: af843ddb-44e1-4e59-8c6d-b09a1ad288a2
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

[root@storage ~]# sudo mkfs.ext4 /dev/mapper/storage-
storage-big_data   storage-smol_data  
[root@storage ~]# sudo mkfs.ext4 /dev/mapper/storage-big_data 
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done                            
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: bd9f1504-c63d-4982-808e-51fd4b885820
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
```
🌞 **Montez la partition**

```bash
[root@storage ~]# sudo mkdir -p /mnt/lvm_storage 
[root@storage ~]# sudo mount /dev/mapper/storage-big_data /mnt/lvm_storage/
[root@storage ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      4.0M     0  4.0M   0% /dev
tmpfs                         385M     0  385M   0% /dev/shm
tmpfs                         154M  3.1M  151M   2% /run
/dev/mapper/rl-root           8.0G  1.5G  6.5G  19% /
/dev/sda1                     960M  311M  650M  33% /boot
tmpfs                          77M     0   77M   0% /run/user/0
/dev/mapper/storage-big_data  2.0G   24K  1.8G   1% /mnt/lvm_storage
```

🌞 **Configurer un *automount***

```bash
[root@storage lvm_storage]# nano /etc/fstab 
[root@storage mnt]# sudo cat /etc/systemd/system/mnt-lvm_storage.mount 
[Unit]
Description=Mount LVM Storage
After=local-fs.target

[Mount]
What=/dev/mapper/storage-big_data
Where=/mnt/lvm_storage
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target
[root@storage mnt]# sudo cat /etc/systemd/system/mnt-lvm_storage.automount
[Unit]
Description=Automount LVM Storage on Demand
After=local-fs.target

[Automount]
Where=/mnt/lvm_storage
TimeoutIdleSec=5min

[Install]
WantedBy=multi-user.target
[root@storage lvm_storage]# sudo systemctl daemon-reload
sudo systemctl enable mnt-lvm_storage.automount
sudo systemctl start mnt-lvm_storage.automount
```



🌞 **Prouvez que l'*automount* est effectif**

```bash
[root@storage mnt]# df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                385M     0  385M   0% /dev/shm
tmpfs                154M  3.1M  151M   2% /run
/dev/mapper/rl-root  8.0G  1.5G  6.5G  19% /
/dev/sda1            960M  311M  650M  33% /boot
tmpfs                 77M     0   77M   0% /run/user/0
[root@storage mnt]# ls
lvm_storage
[root@storage mnt]# cd lvm_storage/
[root@storage lvm_storage]# ls
lost+found
[root@storage lvm_storage]# dh -h
-bash: dh: command not found
[root@storage lvm_storage]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      4.0M     0  4.0M   0% /dev
tmpfs                         385M     0  385M   0% /dev/shm
tmpfs                         154M  3.1M  151M   2% /run
/dev/mapper/rl-root           8.0G  1.5G  6.5G  19% /
/dev/sda1                     960M  311M  650M  33% /boot
tmpfs                          77M     0   77M   0% /run/user/0
/dev/mapper/storage-big_data  2.0G   24K  1.8G   1% /mnt/lvm_storage
[root@storage mnt]# date
Thu Nov 14 14:06:58 CET 2024
[root@storage mnt]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      4.0M     0  4.0M   0% /dev
tmpfs                         385M     0  385M   0% /dev/shm
tmpfs                         154M  3.1M  151M   2% /run
/dev/mapper/rl-root           8.0G  1.5G  6.5G  19% /
/dev/sda1                     960M  311M  650M  33% /boot
tmpfs                          77M     0   77M   0% /run/user/0
[root@storage mnt]# date
Thu Nov 14 14:12:45 CET 2024
```


# III. RAID

Dans cette section, vous allez vous servir des disques supplémentaires ajoutés à la VM.

Installez `mdadm` sur votre VM, on va avoir besoin de lui pour mettre en place un RAID logiciel.

Ici, pas de carte RAID physique (ou virtuelle...) qui gère un RAID bas niveau, c'est un RAID géré par un programme une fois l'OS lancé. L'OS a donc la visibilité sur les disques sous-jacents.

> En effet, dans le cas d'un RAID logiciel, l'OS ne voit qu'un disque unique, qui est en réalité le RAID mis en place par la carte RAID physique.

## 1. Simple RAID

🌞 **Mettre en place un RAID 5**

```bash
[root@storage mnt]# sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdc /dev/sdd /dev/sde
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

🌞 **Prouvez que le RAID5 est en place**

```bash
[root@storage mnt]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[3] sdd[1] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      
unused devices: <none>
```

🌞 **Rendre la configuration automatique au boot de la machine**

```bash
[root@storage ~]# sudo mdadm --detail --scan --verbose >> /etc/mdadm.conf
[root@storage ~]# cat /etc/mdadm.conf 
ARRAY /dev/md0 level=raid5 num-devices=3 metadata=1.2 name=storage.b3:0 UUID=c94d9638:96cf8609:98efbe8b:ef4165ce
   devices=/dev/sdc,/dev/sdd,/dev/sde
[root@storage ~]# sudo dracut --regenerate-all --force
[root@storage ~]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[3] sdd[1] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      
unused devices: <none>
[root@storage ~]# reboot
[root@storage ~]# Connection to 192.168.10.20 closed by remote host.
Connection to 192.168.10.20 closed.
adm_opd@bastion:~$ ssh -p 2222 root@192.168.10.20
root@192.168.10.20's password: 
Last login: Thu Nov 14 14:40:57 2024 from 192.168.10.15
[root@storage ~]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[3] sdd[1] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      
unused devices: <none>
```

🌞 **Créez un système de fichiers sur la partition proposé par le RAID**

```bash
[root@storage ~]# sudo mkfs.ext4 /dev/md0
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2618880 4k blocks and 655360 inodes
Filesystem UUID: ffb6642f-db49-4579-bfc4-7b71aeb9a2ec
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
```

🌞 **Monter la partition sur `/mnt/raid_storage`**

```bash
[root@storage ~]# sudo mkdir /mnt/raid_storage
[root@storage ~]# sudo mount /dev/md0 /mnt/raid_storage
[root@storage ~]# df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                385M     0  385M   0% /dev/shm
tmpfs                154M  3.1M  151M   3% /run
/dev/mapper/rl-root  8.0G  1.6G  6.4G  20% /
/dev/sda1            960M  395M  566M  42% /boot
tmpfs                 77M     0   77M   0% /run/user/0
/dev/md0             9.8G   24K  9.3G   1% /mnt/raid_storage
```

🌞 **Prouvez que...**

```bash
[root@storage raid_storage]# nano toto.txt
[root@storage raid_storage]# ls
lost+found  toto.txt
[root@storage raid_storage]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      4.0M     0  4.0M   0% /dev
tmpfs                         385M     0  385M   0% /dev/shm
tmpfs                         154M  3.1M  151M   2% /run
/dev/mapper/rl-root           8.0G  1.6G  6.4G  20% /
/dev/sda1                     960M  395M  566M  42% /boot
/dev/md0                      9.8G   28K  9.3G   1% /mnt/raid_storage
/dev/mapper/storage-big_data  2.0G   24K  1.8G   1% /mnt/lvm_storage
```
mais disque font 5 Go

🌞 **Mini benchmark**
```bash
[root@storage raid_storage]# sudo dd if=/dev/zero of=/mnt/raid_storage/testfile bs=1M count=1024 oflag=direct
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 3.829 s, 280 MB/s
[root@storage raid_storage]# sudo dd if=/dev/zero of=/mnt/lvm_storage/testfile bs=1M count=1024 oflag=direct
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 2.36083 s, 455 MB/s
```

## 2. Break it

🌞 **Simule une panne**

```bash
[root@storage raid_storage]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sdb                   8:16   0    5G  0 disk  
├─storage-smol_data 253:2    0    2G  0 lvm   
└─storage-big_data  253:3    0    2G  0 lvm   /mnt/lvm_storage
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdd                   8:48   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage 
[root@storage raid_storage]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sdb                   8:16   0    5G  0 disk  
├─storage-smol_data 253:2    0    2G  0 lvm   
└─storage-big_data  253:3    0    2G  0 lvm   /mnt/lvm_storage
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage 
```

🌞 **Montre l'état du RAID dégradé**

```bash
[root@storage raid_storage]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[3] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
      
unused devices: <none>
```

🌞 **Remonte le disque dur**

```bash
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[3] sdd[1] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [======>..............]  recovery = 32.5% (1706068/5237760) finish=0.3min speed=155097K/sec
      
unused devices: <none>
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[3] sdd[1] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [==================>..]  recovery = 93.8% (4915540/5237760) finish=0.0min speed=158565K/sec
```

## 3. Spare disk

Un disque en *spare* c'est un terme qu'on utilise pour désigner un disque qui ne fait rien, mais est prêt à prendre la place d'un disque défaillant.

🌞 **Ajoutez un disque encore inutilisé au RAID5 comme disque de *spare***

```bash
[root@storage ~]# sudo mdadm --add /dev/md0 /dev/sdf
mdadm: Value "storage.b3:0" cannot be set as name. Reason: Not POSIX compatible. Value ignored.
mdadm: added /dev/sdf
[root@storage ~]# nano /etc/mdadm.conf 
[root@storage ~]# cat /etc/mdadm.conf 
ARRAY /dev/md0 level=raid5 num-devices=4 metadata=1.2 name=storage.b3:0 UUID=3ba24d32:d1d39380:30935585:d3e58f1a
   devices=/dev/sdc,/dev/sdd,/dev/sde,/dev/sdf
[root@storage ~]# lsblk
[...]
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS  
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 
sdd                   8:48   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 
sdf                   8:80   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 
sdg                   8:96   0    5G  0 disk  
sr0                  11:0    1  1.7G  0 rom   
[root@storage ~]# cat /proc/
Display all 183 possibilities? (y or n)
[root@storage ~]# cat /proc/m
mdstat   meminfo  misc     modules  mounts   mtrr     
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[4](S) sde[3] sdd[1] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      
unused devices: <none>
```

🌞 **Simuler une panne**

```bash
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[4] sde[3] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
      [==>..................]  recovery = 13.4% (704912/5237760) finish=0.4min speed=176228K/sec
      
unused devices: <none>
[root@storage ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    0   10G  0 disk  
├─sda1                8:1    0    1G  0 part  /boot
└─sda2                8:2    0    9G  0 part  
  ├─rl-root         253:0    0    8G  0 lvm   /
  └─rl-swap         253:1    0    1G  0 lvm   [SWAP]
sdb                   8:16   0    5G  0 disk  
├─storage-smol_data 253:2    0    2G  0 lvm   
└─storage-big_data  253:3    0    2G  0 lvm   
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdf                   8:80   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdg                   8:96   0    5G  0 disk  
sr0                  11:0    1  1.7G  0 rom   
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[4] sde[3] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      
unused devices: <none>
```

🌞 **Remonter le disque débranché**

```bash
[root@storage ~]# sudo mdadm --manage /dev/md0 --add /dev/sdd
mdadm: Value "storage.b3:0" cannot be set as name. Reason: Not POSIX compatible. Value ignored.
mdadm: added /dev/sdd
[root@storage ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    0   10G  0 disk  
├─sda1                8:1    0    1G  0 part  /boot
└─sda2                8:2    0    9G  0 part  
  ├─rl-root         253:0    0    8G  0 lvm   /
  └─rl-swap         253:1    0    1G  0 lvm   [SWAP]
sdb                   8:16   0    5G  0 disk  
├─storage-smol_data 253:2    0    2G  0 lvm   
└─storage-big_data  253:3    0    2G  0 lvm   
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdd                   8:48   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdf                   8:80   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdg                   8:96   0    5G  0 disk  
sr0                  11:0    1  1.7G  0 rom   
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdd[5](S) sdf[4] sde[3] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      
unused devices: <none>
```

## 4. Grow

🌞 **Ajoutez un disque encore inutilisé au RAID5 comme disque de *spare***

```bash
[root@storage ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    0   10G  0 disk  
├─sda1                8:1    0    1G  0 part  /boot
└─sda2                8:2    0    9G  0 part  
  ├─rl-root         253:0    0    8G  0 lvm   /
  └─rl-swap         253:1    0    1G  0 lvm   [SWAP]
sdb                   8:16   0    5G  0 disk  
├─storage-smol_data 253:2    0    2G  0 lvm   
└─storage-big_data  253:3    0    2G  0 lvm   
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdd                   8:48   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdf                   8:80   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdg                   8:96   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sr0                  11:0    1  1.7G  0 rom   
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdg[6](S) sdd[5](S) sdf[4] sde[3] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      
unused devices: <none>
```

🌞 **Grow !**

```bash
[root@storage ~]# sudo mdadm --grow /dev/md0 --raid-devices=4
mdadm: Value "storage.b3:0" cannot be set as name. Reason: Not POSIX compatible. Value ignored.
[root@storage ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
[...]  
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdd                   8:48   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdf                   8:80   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sdg                   8:96   0    5G  0 disk  
└─md0                 9:0    0   10G  0 raid5 /mnt/raid_storage
sr0                  11:0    1  1.7G  0 rom   
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdg[6] sdd[5](S) sdf[4] sde[3] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      [==>..................]  reshape = 10.3% (541412/5237760) finish=1.5min speed=49219K/sec
      
unused devices: <none>
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdg[6] sdd[5](S) sdf[4] sde[3] sdc[0]
      10475520 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      [===================>.]  reshape = 99.8% (5230592/5237760) finish=0.0min speed=64389K/sec
      
unused devices: <none>
```

🌞 **Prouvez que le RAID5 propose désormais 4 disques actifs**

```bash
[root@storage ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdg[6] sdd[5](S) sdf[4] sde[3] sdc[0]
      15713280 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      
unused devices: <none>
[root@storage ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
[...]
sdc                   8:32   0    5G  0 disk  
└─md0                 9:0    0   15G  0 raid5 /mnt/raid_storage
sdd                   8:48   0    5G  0 disk  
└─md0                 9:0    0   15G  0 raid5 /mnt/raid_storage
sde                   8:64   0    5G  0 disk  
└─md0                 9:0    0   15G  0 raid5 /mnt/raid_storage
sdf                   8:80   0    5G  0 disk  
└─md0                 9:0    0   15G  0 raid5 /mnt/raid_storage
sdg                   8:96   0    5G  0 disk  
└─md0                 9:0    0   15G  0 raid5 /mnt/raid_storage
sr0                  11:0    1  1.7G  0 rom 
```

🌞 **Euuuh wait a sec... `/mnt/raid_storage` ???**

```bash
[root@storage ~]# sudo resize2fs /dev/md0
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/md0 is mounted on /mnt/raid_storage; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 2
The filesystem on /dev/md0 is now 3928320 (4k) blocks long.

[root@storage ~]# df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                385M     0  385M   0% /dev/shm
tmpfs                154M  3.9M  150M   3% /run
/dev/mapper/rl-root  8.0G  1.6G  6.4G  20% /
/dev/sda1            960M  395M  566M  42% /boot
/dev/md0              15G   24K   14G   1% /mnt/raid_storage
tmpfs                 77M     0   77M   0% /run/user/0
```

# IV. NFS

Enfin, on clôt le TP avec un **partage réseau simple à setup et relativement efficace : NFS.** Il est assez répandu dans le monde opensource pour des charges faibles.

C'est **un modèle de client/serveur** : on installe un serveur NFS sur une machine, on le configure pour indiquer quel(s) dossier(s) on veut partager sur le réseau, et d'autres machines peuvent s'y connecter pour accéder à ces dossiers.

🌞 **Installer un serveur NFS**

```bash
[root@storage lvm_storage]
sudo chmod -R 755 /mnt/raid_storage
sudo chmod -R 755 /mnt/lvm_storage
[root@storage lvm_storage]# sudo nano /etc/exports
[root@storage lvm_storage]# sudo systemctl enable --now nfs-server
Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /usr/lib/systemd/system/nfs-server.service.
[root@storage lvm_storage]# sudo exportfs -rav
exporting 192.168.10.0/24:/mnt/lvm_storage
exporting 192.168.10.0/24:/mnt/raid_storage
[root@storage lvm_storage]# sudo exportfs -v
/mnt/raid_storage
                192.168.10.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
/mnt/lvm_storage
                192.168.10.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
[root@storage lvm_storage]# sudo firewall-cmd --permanent --zone=public --add-service=nfs
sudo firewall-cmd --permanent --zone=public --add-service=rpc-bind
sudo firewall-cmd --permanent --zone=public --add-service=mountd
sudo firewall-cmd --reload
success
success
success
success
[root@storage lvm_storage]# sudo firewall-cmd --zone=trusted --add-interface=ens18 --permanent
sudo firewall-cmd --reload
The interface is under control of NetworkManager, setting zone to 'trusted'.
success
success
```

🌞 **Pop une deuxième VM en vif**

```bash
[root@git test_raid]# sudo mount -t nfs 192.168.10.20:/mnt/lvm_storage /mnt/test_big/
[root@git test_raid]# sudo mount -t nfs 192.168.10.20:/mnt/raid_storage /mnt/test_raid/
[root@git test_raid]# ls
lost+found
[root@git test_raid]# cd ..
[root@git mnt]# ls
test_big  test_raid
[root@git mnt]# cd test_big/
[root@git test_big]# ls
lost+found  testfile
```

🌞 **Benchmarkz**

```bash
[root@git mnt]# dd if=/dev/zero of=/mnt/test_raid/testfilentfs bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 3.35611 s, 320 MB/s
[root@git mnt]# dd if=/dev/zero of=/mnt/test_big/testfilentfs bs=1G count=1 oflag=direct
dd: error writing '/mnt/test_big/testfilentfs': No space left on device
1+0 records in
0+0 records out
937689088 bytes (938 MB, 894 MiB) copied, 6.0969 s, 154 MB/s
#oups y a plus de place 


[root@storage lvm_storage]# ls
lost+found  testfile  testfilentfs
[root@storage lvm_storage]# cd ..
[root@storage mnt]# ls
lvm_storage  raid_storage
[root@storage mnt]# cd raid_storage/
[root@storage raid_storage]# ls
lost+found  testfilentfs
[root@storage raid_storage]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      4.0M     0  4.0M   0% /dev
tmpfs                         385M     0  385M   0% /dev/shm
tmpfs                         154M  3.1M  151M   3% /run
/dev/mapper/rl-root           8.0G  1.6G  6.4G  20% /
/dev/sda1                     960M  395M  566M  42% /boot
/dev/md0                       15G  1.1G   13G   8% /mnt/raid_storage
/dev/mapper/storage-big_data  2.0G  1.9G     0 100% /mnt/lvm_storage
```