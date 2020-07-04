---
title: "Verify your Kubernetes Cluster Network Policies: From Faith to Proof"
date: 2020-06-27T11:39:55+02:00
draft: false
tags:
- Kubernetes
- security
- Network Policies
---

Implement a technical check that verifies implemented security measurements. In case of network policy, try to establish a blocked network connection. Keep the checks as simple as possible and propagate the results in existing monitoring solution.

<!--more--> 

## Intro

Welcome to me my first blog post. I decided to drop my first letters on one of my favoured research topics: [Kubernetes](https://kubernetes.io/). To be more specific, this blog post addresses network policy verification in Kubernetes. 

It is an ongoing discussion as a security consultant to rate if something "is secure" and to motivate people to trust in the recommended solution that its secure. People always argue and discuss about feelings – something feels not secure // I feel not confi with the solution // are you sure that it is secure // whatever. So, you may now ask yourself how to handle such situations? To be honest, one must find a way to convince people that the introduced approach is "secure". As a technician I have trust in my solutions, but how to convince someone who has no trust?

This blog post addresses continuous verification of introduced security measurements, with [Kubernetes Network Policies](/posts/kubernetes-basics/#network-policies) as an example.

## Technical Background

For a basic understanding, it would be from relevance to have some knowledge in the following topics:

- [Kubernetes Basics](/posts/kubernetes-basics/#kubernetes-itself)
- [Kubernetes Networking](/posts/kubernetes-basics/#networking)
- [Kubernetes Network Policies](/posts/kubernetes-basics/#network-policies)

## Solution

I am absolutely not a mathematician or old Greek, but **reductio ad absurdum** describes how we are going to propose an approach to convert my trust into provable trust. 

### General approach

The easiest way to summarize the solution is the phrase **PoC||GTFO**. But, what does this exactly mean?

Similar to test driven development one can create test cases for infrastructure components that verify that implemented security measurements are sufficient. A lot of people do handle Kubernetes as a magic black box, or at least they speak about it like that. To put light into the dark, a lot of features and functionalities are implemented in basic Unix features – especially in terms of security on container level.

The major question is now – **How do we verify that implemented features are sufficient? Just test it.** Since we know that at the end of the day a lot of security measurements in Kubernetes are implemented by the container runtime environment, we can explicit test for them. To verify that a measurement is sufficient, a technical test can be implemented that focus on the feature. 

Before one create a test, it is necessary to **investigate the implemented measurements and figure out how they are realized**. Based on that knowledge one can **implement a simple check**. The overall approach is to implement a check **based on the KISS principle**. Test for container escapes, test for resource exhaustion or test for possible network communication. Multiple tests can be **performed with simple Unix command line tools** or other quick hacks or code snippets from Stack Overflow ;o). After executing these checks, the program returns with a return code that informs about a success or a failure. Such result can be evaluated, and the status propagated – technically spoke, **check continiously and evaluate the return code**.

The magic question is now, how can we **continuously monitor** if security measurements stays valid? It’s easy – we use the **same approach as the rest of the cluster workload** is monitored. 

If a Kubernetes cluster is used to ship an application, somewhere the health state of the Pods is monitored. Normally, there is a dashboard with fancy pie charts and other useful information and typically there is also a place that shows which containers are up and running. If we want to propagate the **failure** of a **security checks** into the dashboard the easiest way is to let the container **crash continuously**. The status of the crashed container is **propagated into** the normal **container monitoring lifecycle** and follow-up actions must be performed. 

### Technical Implementation

As an example how to create a proof of concept for a security measurement I decided to implement a check for [Network Policies](/posts/kubernetes-basics/#network-policies). What is the easiest way to check if traffic is filter? Right – try to connect to the target should not be available. In this situation we may also be also interested if certain other hosts are available.

That sounds like a small project, which can be implemented in a few lines of code. That is perfect to start learning a programming language like Rust, Go or classy C, right? No – absolutely not in our situation. We need a quick implementable check for our measurement. The check should be easy understandable, implementable, reproducible, and so on. It is just a TCP connection – so I decided to use netcat and some bash and that’s it. The check itself fits into one line:

```bash
echo foo | ncat $TARGET $PORT > /dev/null 2>&1
```

If a connection can be established the command above will be zero. If the command above fails, because a connection cannot be established, the return code is not zero. The evaluation can also be done in one line, as following:

```bash
[[ $? != 0 ]]
```

Combining both commands from above with some shell loops and sleep timers in a [shell script](https://github.com/NodyHub/docker-k8s-resources/blob/master/docker-images/test-tcp/test.sh) and move the shell script in a [Dockerfile](https://github.com/NodyHub/docker-k8s-resources/blob/master/docker-images/test-tcp/Dockerfile), we are good to go to have our continuous security measurement verification in place. As soon the image is build and pushed to a container registry, it can even be executed in a Kubernetes cluster with a respective [deployment](https://github.com/NodyHub/docker-k8s-resources/blob/master/docker-images/test-tcp/test-tcp.yaml). It is even possible to label the deployment in the same manner like a deployed application. This is an opportunity to cross-check implemented Network Policies. Furthermore, Kubernetes allows to inject configuration into container so that the to-be-checked hosts can be adjusted on the fly. 

### Demo

As demo show I have set up a cluster and my security test should ensure that we have access to the internet on port 80 and the MongoDB from my namespace is not accessible on its default port.

An excerpt from the deployed [Configmap](https://kubernetes.io/docs/concepts/configuration/configmap/) configuration would be following:

```yaml
apiVersion: v1
kind: ConfigMap
data:
  allow.lst: |
    1.1.1.1:80
  deny.lst: |
    mongo:27017
```

After deploying the container, we can see in the log output that the checks are successful as desired:

```bash
$ kubectl logs test-tcp-564f4b9b58-spjmp
[25/06/2020 14:52:00 | DEBUG] Check hosts from allow.lst to be available
[25/06/2020 14:52:00 | INFO] 1.1.1.1:80 is accessible
[25/06/2020 14:52:00 | DEBUG] Check hosts from deny.lst to be not available
[25/06/2020 14:52:01 | INFO] mongo:27017 not accessible
[25/06/2020 14:52:01 | DEBUG] wait 60 seconds and repeat
```

To verify that our script is sufficient working we can add another host to the `deny.lst`, which we know that is accessible, but should not. The configmap can be adjusted during usage in the cluster and the updated value is propagated into the pods:

```bash
$ kubectl logs test-tcp-564f4b9b58-spjmp
[…]
[25/06/2020 14:58:02 | DEBUG] Check hosts from allow.lst to be available
[25/06/2020 14:58:02 | INFO] 1.1.1.1:80 is accessible
[25/06/2020 14:58:02 | DEBUG] Check hosts from deny.lst to be not available
[25/06/2020 14:58:02 | INFO] mongo:27017 not accessible
[25/06/2020 14:58:02 | ERROR] 1.1.1.1:80 is accessible, but should not be !!
$
```

As we can see the pod crashed and the cluster always tries to restart the pod. These crashes are propagated into the in-place container monitoring solution. The support or operations team that monitors the cluster health status can now react and create a ticket or even remediate the issue by performing a role-back of previous enrolled changes. The overall status change would have after remediation an output like the following:

```bash
$ kubectl get pods -w
NAME                        READY   STATUS             RESTARTS   AGE
test-tcp-564f4b9b58-spjmp   1/1     Running            0          5m43s
test-tcp-564f4b9b58-spjmp   0/1     Error              0          6m9s
test-tcp-564f4b9b58-spjmp   0/1     Error              1          6m11s
test-tcp-564f4b9b58-spjmp   0/1     CrashLoopBackOff   1          6m26s
test-tcp-564f4b9b58-spjmp   0/1     Error              2          6m27s
test-tcp-564f4b9b58-spjmp   0/1     CrashLoopBackOff   2          6m43s
test-tcp-564f4b9b58-spjmp   0/1     Error              3          6m56s
test-tcp-564f4b9b58-spjmp   0/1     CrashLoopBackOff   3          7m7s
test-tcp-564f4b9b58-spjmp   0/1     Error              4          7m52s
test-tcp-564f4b9b58-spjmp   0/1     CrashLoopBackOff   4          8m6s
test-tcp-564f4b9b58-spjmp   0/1     Error              5          9m23s
test-tcp-564f4b9b58-spjmp   0/1     CrashLoopBackOff   5          9m36s
test-tcp-564f4b9b58-spjmp   1/1     Running            6          12m
```

In my case I performed a role-back of the configuration and the pod could restart and go back into the continuous while loop and stays in the running status.

## Summary

The general solution to generate trust for an implemented security measurement is by **implementing a check** that verifies the assumptions. To make use of the implemented check, it must also **easily integratable into existing life cycle**, e.g., continuous monitoring. 

We have seen an approach how to **use basic Unix command line tools** to verify the validity of an implemented security measurement. The implementation of a technical PoC is kept as simple as possible to reduce bugs in the test itself.

The overall intention is to **reduce the amount of time that is spend during endless meetings that discuss if a measurement is sufficient or not**. Using the same time to **implement a PoC** will reveal in a provable statement, which can be rechecked every now and then. 

## Remarks

I remembered during writing of this blogpost that the topic is highly related to the BlackHat London 2019 talk [Reverse Engineering and Exploiting Builds in the Cloud](https://www.blackhat.com/eu-19/briefings/schedule/#reverse-engineering-and-exploiting-builds-in-the-cloud-17287). The researchers state in their session how security checks can be continuously performed during the build process for the build pipeline. [@brompwnie](https://twitter.com/brompwnie) developed for security checks the tool [Break out the Box (BOtB)](https://github.com/brompwnie/botb), which can perform various security checks continuously.

As a little disclaimer, the implemented bash script is a quick and dirty hack, which should show that not always an over-engineered solution is necessary to perform simple tasks ;o) 

## Links

- [Shell script](https://github.com/NodyHub/docker-k8s-resources/blob/master/docker-images/test-tcp/test.sh)
- [Dockerfile](https://github.com/NodyHub/docker-k8s-resources/blob/master/docker-images/test-tcp/Dockerfile)
- [Docker Hub Image](https://hub.docker.com/r/nodyd/test-tcp)
- [Kubernetes Resources](https://github.com/NodyHub/docker-k8s-resources/blob/master/docker-images/test-tcp/test-tcp.yaml)

