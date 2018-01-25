---
layout: post
title:  "Chef? An introduction to Configuration Management"
date:   2018-01-25 12:03:29 -0700
categories: Intro Chef
---

## Introduction

There comes a time in every Sysadmins life where we get tired of doing the same things over and over...  Typically this manifests initially by acquiring some kind of programming/scripting skill, often in the form of [Bash](https://www.gnu.org/software/bash/), [Perl](https://www.perl.org/), [Ruby](https://www.ruby-lang.org), or [Python](https://www.python.org/).  Everyone has their own opinion as to "best" and "worst" tool...  i.e.: "Perl is a write only language" or "Tabs or spaces?! Why the hell do we use invisible control characters!"; every tool has pros and cons.

Regardless of which tool you chose, the journey often starts out as "hey, if I pipe the output of `ls` to `perl` (or `sed` or `awk`) I can filter my results and do a thing!  That's so much easier than copy/paste!".  Many of you will graduate to massive one liners...

```
git branch -a | grep "remotes/origin" | grep -v master | sed 's/^[ *]*//' | sed 's/remotes\/origin\///' | head -n10 | sed 's/^/git push origin :/' | bash
```

(This deletes the first 10 branches from a remote origin, excluding master...)

But that isn't very maintainable, and probably took a few iterations to get right...

The next natural evolution is a series of scripts.  A prevalent philosophy, and one that I share, is if you do it more than twice, script it.  This often leads to a directory full of one off scripts, tools, etc. written in a plethora of languages, often built by many individuals who have left the company long ago... any of this sounding familiar?

Usually about this time executives are starting to demand more, faster, with less.  Your Sysadmins are bogged down in trying to maintain this pile of disjointed legacy scripts that the original developers have long since sublimated into the ether...  A few of the more enterprising individuals will try to build a framework...

You need _*SOMETHING*_ but what?

## Enter Configuration Management...

If you want to know how Wikipedia defines [Configuration Management](https://en.wikipedia.org/wiki/Configuration_management) you can read about it there.  Go ahead, we can wait.

Back?  OK, great.  Let me paraphrase, at a high level:

_Configuration management is a method for maintaining a consistent, repeatable, and safe process for deploying, upgrading, and decommissioning, services, applications, and servers_.

Put another way, give a human a fool proof procedure, and they'll find a way to muck it up, we need a way to eliminate the human factor.  Computers are really good at doing what they're told.  They don't have opinions, they don't get distracted, they won't argue with you (well, they might, but only if you tell them something incorrect... PC LOAD LETTER!!!)

There are a number of tools out there, again, each with their pros and cons.  We're here to talk about Chef.

## Chef

### A brief, biased, and mostly incorrect history...

In the beginning there was [CFEngine](https://en.wikipedia.org/wiki/CFEngine), written by Mark Burgess based on his [Promise Theory](https://en.wikipedia.org/wiki/Promise_theory) it was a tool significantly ahead of its time.  Because it was so ahead of its time, it failed to get any significant adoption and therefore funding.  A few years after CFEngine3's release,  Luke Kanies releases [Puppet](https://en.wikipedia.org/wiki/Puppet_(software\)) based on many of the same principles as CFEngine and designed by Sysadmins for Sysadmins, to do Sysadmin things (and in my opinion those were _grumpy_ Sysadmins who were working out some issues).  Puppet solves an immediate need, solves problems in a little better manner than CFEngine, and has better funding, so sees immediate adoption and creates buzz around the industry.  There were however, some issues around how Puppet resolves order which could potentially cause race conditions, Adam Jacob raised these errors with a solution and was promptly told "No Thanks", so he went and created Chef.

It wasn't all roses.  There were a number of growing pains, but one thing Chef has always done from the beginning is scale, scale fast, and scale large.  From a resources perspective, you can manage 10, 1000, or 10,000 nodes with the same Chef Server.

### The Difference

For me, the difference has always been the ecosystem and the tools.  We have [ChefSpec](http://chefspec.github.io/chefspec/) for that immediate TDD feedback loop.  What's the best way to see if your code works?  Run it!  To that end we have [Test Kitchen](https://kitchen.ci/), which automates creating an instance, running your Cookbook (in keeping with the motif), and running your automated validation tests.  We leverage [InSpec](https://www.inspec.io/) to do our testing, which we can then also leverage for compliance validation, so we can start validating the security configs/applications/etc earier in the process.  Not to mention [Top Notch Tutorials](https://learn.chef.io/).

## Conclusion

It's time to stop treating servers like pets and time to start treating them like cattle.  And use the cattle to manage the other cattle.  There are a number of amazing tools on the market that solve a multitude of problems.  Find one that solves your problems and you enjoy using.  For me, that is Chef.  So strap in.

- Q


