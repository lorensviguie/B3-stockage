# III. Distributed filesystem

## 1. Master server

🌞 **Installer les paquets nécessaires pour avoir un Moose Master**

```bash
[root@master ~]# sudo yum install moosefs-master moosefs-cgi moosefs-cgiserv moosefs-cli
Last metadata expiration check: 0:02:13 ago on Fri Dec  6 12:04:52 2024.
Package moosefs-master-3.0.118-1.rhsystemd.x86_64 is already installed.
Package moosefs-cgi-3.0.118-1.rhsystemd.x86_64 is already installed.
Package moosefs-cgiserv-3.0.118-1.rhsystemd.x86_64 is already installed.
Package moosefs-cli-3.0.118-1.rhsystemd.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!



```

🌞 **Démarrez les services du Moose Master**

```bash
[root@master ~]# # Start the master server
sudo systemctl start moosefs-master
sudo systemctl enable moosefs-master
sudo systemctl start moosefs-cgi
sudo systemctl enable moosefs-cgi
Created symlink /etc/systemd/system/multi-user.target.wants/moosefs-master.service → /usr/lib/systemd/system/moosefs-master.service.
Failed to start moosefs-cgi.service: Unit moosefs-cgi.service not found.
Failed to enable unit: Unit file moosefs-cgi.service does not exist.

[root@master ~]# sudo systemctl enable moosefs-cgiserv
Created symlink /etc/systemd/system/multi-user.target.wants/moosefs-cgiserv.service → /usr/lib/systemd/system/moosefs-cgiserv.service.
[root@master ~]# sudo systemctl start moosefs-cgiserv
```

🌞 **Ouvrez les ports firewall**

```
[root@master ~]# firewall-cmd --list-all | grep tcp
  ports: 22/tcp 80/tcp 9425/tcp 9419/tcp 9420/tcp 9421/tcp
```

## 2. Chunk servers

🌞 **Installer les paquets nécessaires pour avoir un Moose Chunk Server**

```bash
[root@chunk1 ~]# yum install moosefs-chunkserver
Last metadata expiration check: 0:16:14 ago on Fri Dec  6 13:41:29 2024.
Package moosefs-chunkserver-3.0.118-1.rhsystemd.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
```

🌞 **Modifier la conf du Chunk Server**

```bash
[root@chunk1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.3.250.1  master.tp2.b3
[root@chunk1 ~]# cat /etc/mfs/mfschunkserver.cfg | grep HOST
# BIND_HOST = *
MASTER_HOST = master.tp2.b3
# CSSERV_LISTEN_HOST = *

```

🌞 **Faire appartenir les partitions à partager à l'utilisateur `mfs`**

```bash
[root@chunk1 ~]# ls -al /mnt/
total 24
drwxr-xr-x.   4 root root   44 Dec  6 10:10 .
dr-xr-xr-x.  18 root root  255 Dec  5 12:05 ..
drwxr-xr-x  258 mfs  mfs  8192 Dec  6 13:54 data_chunk1
drwxr-xr-x  258 mfs  mfs  8192 Dec  6 13:54 data_chunk2

```
🌞 **Modifier la conf des disques du Chunk Server**

```bash
[root@chunk1 ~]# cat /etc/mfs/mfshdd.cfg | grep mnt/data
/mnt/data_chunk1
/mnt/data_chunk2
```

🌞 **Démarrez les services du Moose Chunk Server**

```bash
[root@chunk1 ~]# systemctl start moosefs-chunkserver
● moosefs-master.service - MooseFS Master server
     Loaded: loaded (/usr/lib/systemd/system/moosefs-master.service; enabled; preset: disabled)
     Active: active (running) since Fri 2024-12-06 15:02:14 CET; 38min ago
    Process: 2240 ExecStart=/usr/sbin/mfsmaster start (code=exited, status=0/SUCCESS)
   Main PID: 2242 (mfsmaster)
      Tasks: 2 (limit: 11025)
     Memory: 216.6M
        CPU: 9.330s
     CGroup: /system.slice/moosefs-master.service
             ├─2242 /usr/sbin/mfsmaster start
             └─2243 "mfsmaster (data writer)"

Dec 06 15:17:38 master.tp2.b3 mfsmaster[2242]: chunkserver register begin (packet version: 6) - ip: 10.3.250.11 / port: 9422, usedspace: 0 (0.00 GiB), totalspace: 0 (0.00 GiB)
Dec 06 15:17:38 master.tp2.b3 mfsmaster[2242]: connection with CS(10.3.250.11) has been closed by peer
Dec 06 15:17:38 master.tp2.b3 mfsmaster[2242]: chunkserver disconnected - ip: 10.3.250.11 / port: 9422, usedspace: 0 (0.00 GiB), totalspace: 0 (0.00 GiB)
Dec 06 15:17:38 master.tp2.b3 mfsmaster[2242]: server ip: 10.3.250.11 / port: 9422 has been fully removed from data structures
Dec 06 15:19:23 master.tp2.b3 mfsmaster[2242]: chunkserver register begin (packet version: 6) - ip: 10.3.250.11 / port: 9422, usedspace: 0 (0.00 GiB), totalspace: 0 (0.00 GiB)
Dec 06 15:19:23 master.tp2.b3 mfsmaster[2242]: connection with CS(10.3.250.11) has been closed by peer
Dec 06 15:19:23 master.tp2.b3 mfsmaster[2242]: chunkserver disconnected - ip: 10.3.250.11 / port: 9422, usedspace: 0 (0.00 GiB), totalspace: 0 (0.00 GiB)
Dec 06 15:19:23 master.tp2.b3 mfsmaster[2242]: server ip: 10.3.250.11 / port: 9422 has been fully removed from data structures
Dec 06 15:21:26 master.tp2.b3 mfsmaster[2242]: chunkserver register begin (packet version: 6) - ip: 10.3.250.11 / port: 9422, usedspace: 0 (0.00 GiB), totalspace: 0 (0.00 GiB)
Dec 06 15:21:26 master.tp2.b3 mfsmaster[2242]: chunkserver register end (packet version: 6) - ip: 10.3.250.11 / port: 9422
```

## 3. Consume

## 1. Monter la partition Moose

🌞 **Installer les paquets nécessaires pour avoir un Moose Client**

```bash
[root@web ~]# yum install moosefs-client
Last metadata expiration check: 0:02:00 ago on Fri Dec  6 15:38:55 2024.
Package moosefs-client-3.0.118-1.rhsystemd.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
```

🌞 **Monter la partition Moose**

```bash
[root@web ~]# mfsmount /mnt/www/ -H master.tp2.b3
mfsmaster accepted connection with parameters: read-write,restricted_ip,admin ; root mapped to root:root
```

🌞 **Preuve et test d'écriture**

```bash
[root@web mnt]# touch test /mnt/www/
[root@web mnt]# cd 
test  www/  
[root@web mnt]# cd www/
[root@web www]# ls
caca  test.txt
[root@web www]# cat test.txt 
[root@web www]# touch testttttt
[root@web www]# ls
caca  test.txt  testttttt
```

## 2. NGINX

🌞 **Installer et configurer NGINX sur la machine `web.tp2.b3`**

```bash
[root@web www]# dnf install nginx
Last metadata expiration check: 1:20:29 ago on Fri Dec  6 15:38:55 2024.
Package nginx-2:1.20.1-20.el9.0.1.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!

```

🌞 **Démarrer le service NGINX**

```bash
[root@web www]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Fri 2024-12-06 16:56:19 CET; 2min 52s ago
    Process: 1332 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1333 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1334 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1335 (nginx)
      Tasks: 2 (limit: 11025)
     Memory: 2.0M
        CPU: 47ms
     CGroup: /system.slice/nginx.service
             ├─1335 "nginx: master process /usr/sbin/nginx"
             └─1336 "nginx: worker process"

Dec 06 16:56:19 web.tp2.b3 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Dec 06 16:56:19 web.tp2.b3 nginx[1333]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Dec 06 16:56:19 web.tp2.b3 nginx[1333]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Dec 06 16:56:19 web.tp2.b3 systemd[1]: Started The nginx HTTP and reverse proxy server.

```

🌞 **Prouvez avec un `curl` que le site est actif**

```bash
[root@web www]# cat ./index.html
<h1>DUCK SUPREMACY </h1>
root@master ~]# curl -v web
*   Trying 10.3.250.101:80...
* Connected to web (10.3.250.101) port 80 (#0)
> GET / HTTP/1.1
> Host: web
> User-Agent: curl/7.76.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.20.1
< Date: Fri, 06 Dec 2024 15:56:37 GMT
< Content-Type: text/html
< Content-Length: 25
< Last-Modified: Fri, 06 Dec 2024 15:56:14 GMT
< Connection: keep-alive
< ETag: "67531e9e-19"
< Accept-Ranges: bytes
< 
<h1>DUCK SUPREMACY </h1>
* Connection #0 to host web left intact
```

