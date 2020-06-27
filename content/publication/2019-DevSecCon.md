---
title: "How to Secure OpenShift Environments and What Happens If You Don´t"
draft: false
date: "2019-11-20"
description: Conference content from DevSecCon 2019
tags:
- OpenShift
- Kubernetes
- devseccon
- security
---

Jan Harrie, DevSecCon, London, United Kingdom
<!--more-->

## Abstract

OpenShift by Red Hat is one of the major Platform as a Service (PaaS) solutions on the market. It is used to automatically deploy Kubernetes clusters and provides useful extensions for cluster management mixed with some magic under the hood. 
Instantiating a Kubernetes cluster is often a crucial step in setting up a modern application stack. But be aware – a lot of configuration parameters are awaiting you. And here several misconfigurations may occur that can lead up to a compromise of the cluster. Privileged containers, tainting of masters and executing workloads on them, missing role-based access controls, and misconfigured Service Accounts are part of the problem. 
In this talk, I will explain which configuration parameters of an OpenShift environment are critical to ensure the overall security of the deployed Kubernetes clusters. Implications of misconfigurations will be demonstrated during live demos. Finally, recommendations for a secure configuration are provided.


## Media

- [Slides](../201911_DSC_OpenShift_v1.0_export.pdf) or [Google Slides](https://cutt.ly/HeNpUtD)
- [Recording](https://www.youtube.com/watch?v=pa4uZICJRRA)
- [Blogpost](https://insinuator.net/2019/11/dsc19-openshift/)


