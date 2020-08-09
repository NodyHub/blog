---
title: "Pentesting Cloud Sandboxes in the wild"
date: 2020-08-01T15:11:49+02:00
draft: true
tags:
- Docker
- Container
- breakout
- security
---

Matthias Luft & Jan Harrie, Bsides Munich, Munich/Virtual, Germany
<!--more-->

## Abstract

Building on last year’s explanation of container workings under the hood ([Fucking Containers - how do they work?](https://2019.bsidesmunich.org/talks/01-03_Fucking-Containers/)), we explain several techniques for breaking out of misconfigured containers/container hosts. We will discuss the most common misconfigurations (such as extensive container privileges, exposed network services, mounted sockets, internal cluster privileges) and how to test for them. For each discussed attack vector, we will show how it can be automated (and integrated into build pipelines) using a tool of choice. Finally, a comparison of the well known container execution platforms (AWS, Azure, fly.io, GCP, Heroku) will be presented.

## Outline

 - Short Container Re-Cap (make sure to be familiar with [Fucking Containers - how do they work?](https://2019.bsidesmunich.org/talks/01-03_Fucking-Containers/))
 - Attack Vectors
   - Container Privileges
   - Network Services (Generic, Cloud, Cluster)
   - Cluster Privileges
   - Sockets
 - Testing how-to with botb and amicontained
 - Cloud Platform comparison 
 - Conclusions

## Media

- [Slides](../202008_bsides_muc.pdf) or [Google Slides](https://gurke.io/bsidesmuc2020)
- [Material](https://github.com/NodyHub/bsidesmuc2020)
- [Recording]()
- [Blogbost]()


