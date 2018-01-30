---
layout: post
title:  "Chef Getting Started"
date:   2018-01-25 12:03:29 -0700
categories: start
---

## Introduction

There comes a time in every SysAdmin's life where we get tired of doing the same things over and over...  Typically this manifests initially by acquiring some kind of programming/scripting skill, often in the form of [Bash](https://www.gnu.org/software/bash/), [Perl](https://www.perl.org/), [Ruby](https://www.ruby-lang.org), or [Python](https://www.python.org/).  Everyone has their own opinion as to "best" and "worst" tool...  i.e.: "Perl is a write only language" or "Tabs or spaces?! Why the hell do we use invisible control characters!"; every tool has pros and cons.

Regardless of which tool you chose, the journey often starts out as "hey, if I pipe the output of `ls` to `perl` (or `sed` or `awk`) I can filter my results and do a thing!  That's so much easier than copy/paste!".  Many of you will graduate to massive one liners...

```
git branch -a | grep "remotes/origin" | grep -v master | sed 's/^[ *]*//' | sed 's/remotes\/origin\///' | head -n10 | sed 's/^/git push origin :/' | bash
```

(This deletes the first 10 branches from a remote origin, exclding master...)

But that isn't very maintainable, and probably took a few iterations to get right...

The next natural evolution is a series of scripts.  A prevalent philosophy, and one that I share, is if you do it more than twice, script it.  This often leads to a directory full of one off scripts, tools, etc. written in a pleathora of languages, often built by many individuals who have left the company long ago... any of this sounding familar?

Usually about this time executives are starting to demand more, faster, with less.  Your SysAdmins are bogged down in trying to mantain this pile of disjointed legacy scripts that the original developers have long since sublimated into the ether...  A few of the more enterprising individuals will try to build a framework...

You need _*SOMETHING*_ but what?

## Enter Configuration Management...

If you want to know how Wikipedia defines [Configuration Management](https://en.wikipedia.org/wiki/Configuration_management) you can read about it there.  Go ahead, we can wait.

Back?  Ok, great.  Let me paraphrase, at a high level:

_Configuration management is a method for mantaining a consistent, repeatable, and safe process for deploying, upgrading, and decomissioning, services, applications, and servers_.

Put another way, give a human a fool proof procedure, and they'll find a way to muck it up, we need a way to eliminate the human factor.  Computers are really good at doing what they're told.  They don't have opinions, they don't get distracted, they won't argue with you (well, they might, but only if you tell them something incorrect... PC LOAD LETTER!!!)

There are a number of tools out there, again, with their Pros and Cons.

private gist?

{% gist qubitrenegade/cabe8d6c4fba63a8084bf67a5409aca0 %}

This thing on?
