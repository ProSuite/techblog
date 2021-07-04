---
layout: post
title: The case for Quaestor - Load balancing and cluster management.
author: ema
date: 2021-07-04
---
# Requirements

Once a microservice infrastructure is actually implemented in production a set of real-world problems come up that must be solved: Load balancing, health checking and the management of the distributed services and processes on multiple computers.

There are highly sophisticated tools that help running data centers and very large microservice infrastructure, such as Istio or Consul. These tools are tremendously powerful in managing service meshes. However, they might not be applicable in every scenario. In our case, we have a set of services that need high isolation from each other, only a few machines and non-containerized software running on windows servers using .NET 4. In addition we had specific requirements on the load balancing containing some domain logic rather than the often used Layer 4 load balancers. So we wanted to build a simple system that does all that but that can be easily adapted to our needs.

So in short, we need

- A cluster manager that performs the heart beat to check that all specified nodes are up and healthy. It shall re-start any node that is not running by simply re-starting the process.
- A look-aside load balancer that knows the health state and the load of the individual nodes and provides the host name and port of the least busy (or otherwise most appropriate) node to the asking client.

# A simple distributed system

This title sounds like a contradiction in itself. If you want to build a distributed system, [this 20 video MIT lecture](https://www.youtube.com/playlist?list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB) is a highly recommended starting point. However, we are humble application programmers that are mostly overwhelmed by the challenges of our daily life already.

So we needed to build on top of something flexible that works rather than re-invent the hard part. [Etcd](https://etcd.io/) is coming to the rescue. Etcd is a distributed key-value store that handles the hard and scary stuff, such as leader elections and split brains and other things we don't even want to know too much about.

Etcd has simple-to-use endpoints in gRPC that wrap the basic put and get. It also comes with a command line client that helps in monitoring and configuring the etcd service process.

We have built our service registry as an Etcd key-value store. Both the load balancers and the cluster manager (during the heart beat operation) can access the list of services both on local and remote machines, check their health and act accordingly. For the look-aside load balancer this means asking for a load report of each service and serve the address that has the least amount of load at the particular moment. For the cluster manager this could mean stopping (or killing) an unresponsive process and re-starting it.

Using the simple [dotnet-etcd](https://github.com/shubhamranjan/dotnet-etcd) API we could implement our domain logic in a few days while not being encumbered by the complexity of the underlying distributed system on which our service registry is based. Nevertheless the system will be able to scale up over time with more and more servers. All load balancers have simple access to all microservices and the cluster manager can ensure that all the registered services can be reached and report their health status. The clients just know the list of load balancers which will be contacted in a round-robin fashion in case one of them is not available (e.g. due to a service downtime).

Using Etcd is another example of our best-in-class approach for using great, established tools and frameworks that do specific things very well. This leaves us with the implementation of the domain logic in exactly the ways that serve our customers best. It is still not trivial to set up the Etcd configuration correctly for TLS and logging. But we believe that this approach is certainly superior to the introduction of big complex systems that require a lot of training and perform many tasks that are not actually needed. On the other hand, it is obviously not feasible to re-implement the problem of distributed computing for the mentioned requirements. Our current approach leaves us with more time and energy to solve the hard problems that lie at the heart of our domain, such as geometric operations and the verification of geospatial data.

A final note: The consul was the highest office in the roman administrative hierarchy. The quaestor on the other hand was the lowest ranking position with the least amount of power. Hence the name of the [quaestor-mini-cluster](https://github.com/ProSuite/quaestor-mini-cluster) repository.