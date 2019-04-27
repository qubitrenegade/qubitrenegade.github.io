---
layout: post
title:  "Habitat and Consul: Client Mode and Registering Services"
date:   2018-06-05 23:29:47 -0700
categories: Habitat Consul Docker
---

## Previously on "As the QubitRenegade turns..."

* [Consul](https://www.consul.io) an OpenSource(ish, there's an enterprise version that has some awesome features) Distributed, Highly Available, Service Discovery, Health Checking, KV Store, that provides federated Multi Datacenter support.  (disclaimer: I am in absolutely no way, shape, or form associated with HashiCorp.  All opinions are my own and not influenced by schwag.  I just happen to think they make a pretty swell product... even if it is written in go ;) )

* [Habitat](https://www.habitat.sh/learn/) an OpenSource, Artifact Builder, Artifact and Service Manager, Docker/Kubernetes/Serverless Enabler, sane Docker container builder system, with Distributed Service Discovery and "build once run anywhere (no really)" mentality.

* [Docker](https://www.docker.com/) is how most people do containers. Just... I'll post the opinion piece soon.  Docker is fine.

## Prerequisites

* [Habitat](https://www.habitat.sh/docs/install-habitat/)
* [Docker](https://docs.docker.com/install/)

## High level Overview

In the last installment, we exported a Core habitat package to a Docker container, then started several instances to build a cluster leveraging Habitat's service discovery.  To follow along you don't necessarily need all five servers, a single node running in dev mode would be sufficient.  We'll proceed as though you have a full cluster to work with.

Today we'll explore running Consul in "client" mode.

## Justification

Initially, I wanted to have the ability to run `core/consul` in either "Client" or "Server" mode.  The primary difference is that Consul "Servers" participate in gossip and present a Web UI.  Technically the client could present a Web UI.  When running other services (Vault, Nomad, etc. which we'll touch on in a later post), HashiCorp recommends running Consul in client mode locally and not allowing the tools to communicate directly with the consul server.

When discussing Consul, we speak about "Server" and "Client" mode; even though it's the same binary run with different CLI parameters, "client" and "server" makes a natural demarcation which lends itself to the segregation of packages.

## Security

It's worth noting that Habitat and Consul both have methods of encrypting communication.  For the time being this guide is putting that aside, with a note that we will cover it eventually.


## Building Consul Client

The Habitat plan source can be found [on github](https://github.com/qubitrenegade/habitat-consul-client)

If you want to just run it, skip to Exporting Consul Client below.



#### Service Config Files

To enable what is essentially "arbitrary" configuration files, we can use the `toJson` helper method to turn the `default.toml`/`user.toml` config into JSON structures.  As Handlebars is fairly limited, there is no "else if" clause, we generate correct JSON and allow Consul to complain about the configuration file.

```
{
  {{ #if cfg.service }}
    "service": {{toJson cfg.service}}{{#if cfg.services}},{{/if}}
  {{ /if }}
  {{ #if cfg.services}}
    "services": {{toJson cfg.services}}
  {{ /if }}
}
```

This will allow a minimal configuration such as:

```
[service]
name = "foo"
[service.checks]
interval = "1s"
```

Which will generate the JSON:

```
{
  "service": {
    "name": "foo",
    "checks": [
      {
        "interval": "1s"
      }
    ]
  }
}
```

(just not pretty formatted)

## Exporting Consul Client

The Consul Client package is really designed as a building block for other packages that need to leverage a Consul Client.  For demonstrative purposes, we're just going to run Consul by itself.

#### Enter Studio

If you're not in a Linux environment, you'll need to enter the habitat studio:

```
$ hab studio enter
```

#### Export `qubitrenegade/consul-client`

Run the `hab pkg` command to export `qubitrenegade/consul-client` to a docker container.

```
$ hab pkg export docker qubitrenegade/consul-client
```

This will create a directory `$(pwd)/results` and a file `$(pwd)/results/last_docker_export.env` which exports a number of environment variables if sourced.  This enables such idioms as `docker run -it $name`.

We will specify the origin and package name in examples below.

#### Run your docker containers:

You will either need to open multiple terminal windows, or replace the `-e` with `-de`

This also assumes you have no more than one (1) running container before starting this process.  If you have more than one (1) running container, adjust your IPs accordingly.

Create a service config file and save it as `services.json`:

```
{
  "client": {
    "enable_script_checks": true
  },
  "service": {
    "name": "simple-service",
    "checks": [
      {
        "name": "is_alive",
        "args": ["/bin/bash", ":(){ echo "I'm alive!"; return 200; }; :"],
        "interval": "1s"
      }
    ]
  }
}
```

Then start your docker container while importing your settings (adjust peer IPs as necessary):

```
docker run -e "HAB_CONSUL_CLIENT=$(< services.json)" qubitrenegade/consul-client --peer 172.17.0.4 --bind consul-server:consul.default
```

Now when visit [http://localhost:8500](http://localhost:8500) You should see your service registered!

#### Examples

A number of service examples are provided in the [example](https://github.com/qubitrenegade/habitat-consul-client/tree/master/example) directory.

To use any of the examples, you can import the config values as environment variables:

```
docker run -e "HAB_CONSUL_CLIENT=$(curl https://raw.githubusercontent.com/qubitrenegade/habitat-consul-client/master/example/service-with-no-checks.json)" qubitrenegade/consul-client --peer 172.17.0.4 --bind consul-server:consul.default
```
