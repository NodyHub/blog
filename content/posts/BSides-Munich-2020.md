---
title: "Pentesting Cloud Sandboxes in the wild"
date: 2020-08-22T17:07:28+02:00
draft: true
---

Matthias and I talked about cloud sandboxes on virtual BSides Munich 2020. This blogpost summarizes the content of the talk.

<!--more-->

First, I will shout out to the BSides Munich Orga team to make the conference happen. Matthias and I were thankful to have the pleasure to contribute to the event. Furthermore, another thank you to the other speakers, which shared their research during this wired virtual-conference-times!

So, what about the content. We started our talk with a short re-cap about containers, based on the amazing talk “Fucking Containers – how do they work?” from Andreas Krebs which was presented on BSides Munich 2019 ([Ref](https://2019.bsidesmunich.org/talks/01-03_Fucking-Containers/), [Slides](https://raw.githubusercontent.com/BSidesMUC/BsidesMunich2019/master/files/01-03_Fucking-Containers.pdf)). 

![A. Krebs F***ing Containers](../../images/akrebs-container.png)
 

After the re-cap and disclaimer, we start to go through well-known container breakout techniques. To make a long story short, we start with a **root filesystem**, which is **accessible from a container**. In that case, you have to **leverage basic Linux privileges escalation techniques**, by abusing services from the host system. 

![Attack Container Filesystem](../../images/Attacking_Container_filesystem.png)
 

Afterward, we address the configuration issue, which may occur if you have access to the Linux kernel by a **writable `/sys` directory**. In that way, you can **create a call-back** script that is **triggered if a device is plugged** into the system – which can also be simulated. Next breakout would be by **exploitation** of the existence of the **capability `CAP_SYS_MODULE`** to **load kernel modules** and the **capability `CAP_SYS_ADMIN`** to leverage **Linux control groups (cgroups)** to get out of the box. If you have **access to the devices `/dev`** of the container host, you can go and **mount** the **hard drive** and access the root filesystem, as explained prior. And we give a quick reference to what could be done with the `/proc/keys`. 

![Attack Container Hardware and Kernel](../../images/Attacking_Container_hardware-kernel.png)
 

The last technique, which seems to be very local-docker-installation related is the container with **access to the Docker socket**. With access to the Docker socket, you can **start** further **container with “super-powers”**, which allows you previously explained techniques. 

![Attack Container ctrl plane](../../images/Attacking_Container_socket.png)
 

A detailed explanation of these attacks can be found on this blog [Container Breakouts – Part 1](https://blog.nody.cc/posts/container-breakouts-part2/), [Container Breakouts – Part 2](https://blog.nody.cc/posts/container-breakouts-part2/) and [Container Breakouts – Part 3](https://blog.nody.cc/posts/container-breakouts-part2/). You may now think “why do they include the Docker socket within the breakout list?”. That’s the point. Nowadays, a cloud container runtime may come in various flavors. You never know if the container is started within a Kubernetes environment or own-created orchestration layer. We will finally address the container breakout by the usage of the metadata API. The metadata API of the cloud is the control plane of the cloud, like the docker socket is the control plain of the local docker installation.

After discussing the various breakout techniques, we go through a short comparison from wide-spread cloud container runtimes. (We decided to let Kubernetes runtimes out of the comparison). The following screenshot shows the overview table.

![Cloud Platform Comparison](../../images/plattform-comp.png)
 

The data is measured with [botb](https://github.com/brompwnie/botb) and [amicontained](https://github.com/genuinetools/amicontained/). All technical details can be found on [Github](https://github.com/NodyHub/bsidesmuc2020), including the scan results of both tools.

After summarization of the attack vectors, we go through the details of how [botb](https://github.com/brompwnie/botb) and [amicontained](https://github.com/genuinetools/amicontained/) can be used by you, maybe in the next assessment? 

![BotB usage](../../images/botb.png)
 

With the knowledge about the attacks, we go through the mitigation techniques that can be used to prevent breakouts and tune the security of your sandbox. Finally, a conclusion and an outline with further work close the session.

We hope that you enjoyed the session and if you are interested to discuss content, do not hesitate and get in touch.

Cheers,

[Matthias](https://twitter.com/uchi_mata) and [Jan](https://twitter.com/NodyTweet)

-- 

- [Slides](https://gurke.io/bsidesmuc2020/)
- [Recording]()
- [Scan results on GitHub](https://github.com/NodyHub/bsidesmuc2020)
- [BSides Munich](https://www.bsidesmunich.org/)


