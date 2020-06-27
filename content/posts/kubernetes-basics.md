---
title: "Kubernetes Basics and Principles"
draft: false
tags:
- Kubernetes
---

This blogpost is intended to be a background knowledge section for further posts. The content is all around Kubernetes basic principles and reflects more a dictionary then a legitim blog post. The section itself may not explain every nuance of the topic – only as much as necessary as needed in other posts. 

<!--more--> 

The list of topics and explanations may be updated over the time.

## Kubernetes itself
[Kubernetes](https://kubernetes.io/) is a production-grade container orchestration solution that is available on all modern cloud provider for executing and orchestrating containerized applications. The applications are distributed over multiple nodes, which execute the application containers. The container itself are executed by the installed container runtime interface, while the management of the containers is handled by Kubernetes. 

Kubernetes offers fine-grained permission management capabilities that can be set for user and groups – implemented by [Role Based Access Control (RBAC)](). The scope can be set to the overall cluster or only to a [Namespace](). A best-practice to develop a cluster application is to assign projects to its own or multiple Namespaces and hand over the project team administrative access in the development environment. The administrator can manage service accounts, roles, storage, services, pods and further namespace scoped resources. More details about Kubernetes can be read in the article [What is Kubernetes?](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/). 

The most interesting part is that an administrator can install and configure the cluster and slice it into pieces, which can be managed within these pieces independent.

## Networking
One of the Kubernetes advantages (maybe also disadvantages) is that it is built like a big and flexible framework. Besides different storage classes and different authentication provider, the implementation of the [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/) can be exchanged. Kubernetes offers the flexibility to set for the overall cluster one network plugin. These network plugins are created and maintained independent of the Kubernetes project, like [Calico](https://www.projectcalico.org/) or [Cilium](https://cilium.io). Even the cloud provider like AWS implement their own plugins for an optimal integration into the cloud environment. The decision which network plugin depends on the requirements. Requirements can be reliability, encryption, speed, or Network Policies. Only one network plugin can be configured for a cluster.
## Network Policies
[Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) are the kind of resources in Kubernetes that can be used to define stateless firewall rules for managed cluster workload. The policies are applied on Pod’s, namespaces or a combination of namespaces and pods. 

Network Policies are defined on an allow-listing approach and deployed in a namespace. The policy starts with a pod selection – based on the label – on which pod’s it should be applied. Hence, the same labels can be set to different pods, a group selection is also possible. If the Network Policy should be valid for every resource, a wild-card selector (`{}`) can even be used. This selection defines on which resource the rule should be applied. The next part is if it is an ingress or an egress rule and which ports ([OSI layer 4](https://en.wikipedia.org/wiki/OSI_model#Layer_4:_Transport_Layer)) are allowed – the port is optional. Finally, the rule defines the network communication opposite.  The opposite is a pod, a namespace, a combination of pod and namespace or IP block in CIDR notation. Pod and namespace are selected based on the labels, which must have been set previously. If no Network Policy is configured for a pod, the pod can communicate over the network without any restriction.

A best practice approach is to deploy a deny all network policy and configure in an explicit allow-listing approach intended communication paths. But, be aware, this may escalate quickly 😉

## Label and Selectors
[Label and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) is the concept in Kubernetes, which is used to perform group operations on similar resources 

Before a resource can be selected, these resources must be labelled. Kubernetes offers the feature to label each resource with key value pairs. These key value pairs can be selected freely, e.g., `foo: bar`, `stage: dev` or `app: frontend`. Labels can be set to almost every object (not sure if it is possible to label all resources). The semantic of these labels is only in the interpretation of the people that set these labels. Okay, nowadays there might be technical resources that depend on such labels to make decisions. Objects from one type can be selected by selectors. These selectors are analogous to the labels and must match labels that have been previous set, e.g., `foo: bar`, `stage: dev` or `app: frontend`.



