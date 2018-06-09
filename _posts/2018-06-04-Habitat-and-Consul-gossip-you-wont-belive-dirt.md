---
layout: post
title:  "Habitat & Consul: Gossip: You won't believe the dirt!"
date:   2018-06-04 11:29:47 -0700
categories: Habitat Consul Docker
---

## Previously on "As the QubitRenegade turns..."

* [Consul](https://www.consul.io) an OpenSource(ish, there's an enterprise version that has some awesome features) Distributed, Highly Available, Service Discovery, Health Checking, KV Store, that provides federated Multi Datacenter support.  (disclaimer: I am in absolutely no way, shape, or form associated with HashiCorp.  All opinions are my own and not influenced by schwag.  I just happen to think they make a pretty swell product... even if it is written in go ;) )

* [Habitat](https://www.habitat.sh/learn/) an OpenSource, Artifact Builder, Artifact and Service Manager, Docker/Kubernetes/Serverless Enabler, sane Docker container builder system, with Distributed Service Discovery and "build once run anywhere (no really)" mentality.

* [Docker](https://www.docker.com/) is how most people do containers. Just... I'll post the opinion piece soon.  Docker is fine.

## Prerequisites

* [Habitat](https://www.habitat.sh/docs/install-habitat/)
* [Docker](https://docs.docker.com/install/) 

You'll need to install them both.  Basically we're going to run a bunch of containers instead of a bunch of servers.  If we were going to push this into production, it doesn't really make sense to run three-five instances of a service on on VM... if that VM dies, then there goes your cluster.  But with Habitat we can basically run the service in any format we want, containers the same way we'd run them if they were individual VMs or being published to a Kubernetes cluster.  Kubernetes expects you to show up to the party with containers ready to go, Habitat is the one who comes up with the kegs in the trunk.

## High level Overview

Habitat needs at least three nodes to form consensus and elect a "leader".  This prevents "split brain" where two nodes both think they are the leader.  Clusters then scale to uneven numbers... 3, 5, 7, etc.  
Consul needs at least three nodes to form consensus and elect a leader... (sound familiar)...

It's definitely easier to bootstrap a Habitat cluster than a Consul cluster.  By leveraging Habitat we can bootstrap a consul cluster more easily...  This is for sure not a "Habitat vs Consul" this is 100% a "Habitat + Consul", I think they are the perfect compliment for each other.

## Building Consul

You have multiple choices actually.  That's the whole point!  

If you have three (or five!) VMs to run on, you can just install hab on each VM then run:

```
hab sup run core/consul --peer <node X-1> --peer <node X+1>
```

You probably don't want to do this for production, but you get the idea, you can run the packages natively in Linux or in a container on any other platform.

On to the task at hand:

#### Enter Studio

If you're not in a Linux environment, you'll need to enter the habitat studio:

```$ hab studio enter
```

#### Export `core/consul`

Run the `hab pkg` command to export `core/consul` to a docker container.

```$ hab pkg export docker core/consul
```

This will create a directory `$(pwd)/results` and a file `$(pwd)/results/last_docker_export.env` which exports a number of environment variables if sourced.  This enables such idioms as `docker run -it $name`.

We will specify the origin and package name in examples below.

#### Run your docker containers:

You either need to open three or five terminal windows, or replace the `-e` with `-de` in the examples below.

This also assumes you have no more than one (1) running container before starting this process.  If you have more than one (1) running container, adjust your IPs accordingly.

Run once:

```
docker run -e HAB_CONSUL='{ "server": { "beta_ui": true, "bind": "0.0.0.0" }, "client":{"bind": "0.0.0.0"}}' -p 8500:8500 -p 8600:8600/udp core/consul --peer 172.17.0.3 --topology leader --peer 172.17.0.4
```

Run thrice (well, 2 or 4 times but actually not specifically not 3 times)

```
docker run -e HAB_CONSUL='{ "server": { "beta_ui": true, "bind": "0.0.0.0" }, "client":{"bind": "0.0.0.0"}}' core/consul --peer 172.17.0.3 --topology leader --peer 172.17.0.4
```

Now visit [http://localhost:9500](http://localhost:9500)

you should see 5 nodes!
