---
layout: post
title:  "Chef - Inversion of Control/Responsibility: letting other sweat the details"
date:   2019-09-15 12:13:57 -0700
categories: Chef IoC
---

## Introduction

~~We've been running into a situation where there is supporting software required for our application deployment.  For instance, a PHP application requires a web server of some sort.  Hopefully we can make the web server a requirement, but as is often the case, we will need to support apache, nginx, etc.~~

~~This is often solved by "MyApplication" cookbook providing recipes for each of the various web servers as well as the application itself.  This means that the cookbook author (often the application author) is writing recipes for~~

< some situation where this would come into play >

basically want to illustrate how you might run an application in different ways.  And as the cookbook author you might get bogged down in things that are less relevant to your project and/or something you can't/won't test.

< / some situation ...>

Let's look at `certbot`.  [`certbot`]() is a tool to interact with [Let's Encrypt]() to automate the generation and renewal of SSL Certificates.  It does so by automating the validation of the ownership of the domain.  And it does _that_ by making use of "authentication plugins".  These authentication plugins all take different cli parameters from each other meaning the parameters passed to `certbot` will be different depending on the authentication plugin.  For instance, the [certbot-cloudflare-dns](https://certbot-dns-cloudflare.readthedocs.io/en/stable/) plugin takes `--dns-cloudflare-credentials` and the [certbot-dns-digitalocean](https://certbot-dns-digitalocean.readthedocs.io/en/stable/)(hooray for consistent naming standards!) plugin takes `--dns-digitalocean-credentials` parameter.

<!-- [(ED: ok, maybe this is a bad use case and a better way would have just been to implement all of the different plugins if they all have the same parameters...)] -->

There's a [dozen](https://certbot.eff.org/docs/using.html#dns-plugins) dns plugins.  I don't care about most of those and won't test all of the different options.  Not only that but the various different plugins require different packages to be installed.

But regardless of which dns plugin we use with `certbot`, how we interact with `certbot` really doesn't change.  For instance, we're _always_ going to give it `-d` with a list of domains.

It would be really nice for the cookbook author to be able to just call our `certbot_exec` resource, and the consumer of their cookbook be able to use any of DNS plugins.  Ideally, just by including the appropriate plugin cookbook.

## Glossary

Before we really get into it, let's define some terms.

* Resource Cookbook - provides a custom resource but does not provide any runnable recipes.  (Often the `default` recipe will write a log message stating as such or in most severe cases actually aborting the Chef run)
* Plugin Cookbook - modifies the behavior of another cookbook simply by being included in the `metadata.rb`.  Should not be interacted with directly.
* Application Cookbook - leverages any number of resources to build an application.  Only one of the four types that generally has recipes.
* Wrapper Cookbook - wraps other cookbooks to modify their configuration for specific deployment instance.  Generally only attributes and does not have own recipes.

## Soft Theory

By writing our cookbooks, and really, more specifically our resources, in such a way that lends them to being easily extended, we can in effect kick the proverbial can of responsibility up the stack to the "integrator" e.g.: the point where individual building blocks start being combined into more complicated systems.

The idea being that the author of the `certbot_exec` resource is the expert on `certbot`, the author of the `certbot-exec-cloudflare` plugin is the expert on CloudFlare, etc.  and by allowing the experts on these topics to focus they write better code.

## Drawbacks

You can't use two DNS plugins together.  This will cause `certbot` to fail.  Because the `certbot_exec` resource knows nothing about the "plugin cookbook"
