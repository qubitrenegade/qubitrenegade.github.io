---
layout: post
title:  "Building your own Kernel from Source"
date:   2019-08-20 22:38:21 -0700
categories: matrix synapse riot privacy chat
---

It [appears](https://fedoraproject.org/wiki/Building_a_custom_kernel) there is a number of ways to build the kernel.  `fedpkg` doesn't seem to have a `menuconfig` option, so I chose to build against a "vanilla" kernel checked out from _the_ source.

Fortunately, we're able to use our existing kernel config and "upograde" it using `make` (specifically `make olddefconfig`, `yes '' | make oldconfig` for my veterans).

As of this writing, 5.2.8 is the most current 5.2 branch.  Current stable branches can be found in the [linux git repo](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/) ([<-- CLICK HERE for a link to the Linux git repo](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/)).  They are denoted as `<major version>.<minor version>.y` where `y` represents a variable patch level (I think, I would love to be able to link to the kernel docs that describe this naming convention).  You can check out a specific version following this format.  For example if version 6 of the kernel arrives, the `git clone` command would use `-b linux-7.5.y`.

I'm like... 95% sure you don't need fedpkg to build from "Raw" source.  I wonder if there's "Fedora Specific" patches I'm missing out on pulling for git.kernel.org instead of using `fedpkg`...  Now that I basically know what things flags I want to set for the kernel build, I wonder if I can use my `.config` as a way to populate `fedpkg` configs...

> You can now make whatever changes / customizations you need before generating the rpms and installing them. You may want to consider uncommenting 

Make them where?  I usually make them in my `.config` file...  with `make menuconfig`!

> If there are configuration options that need to be adjusted for your build, you can add changes in the kernel-local file. These changes will get picked up when you build. 

Make them how?  Can I `mv .config kernel-local`?  What _IS_ the "kernel-local" file?

I'd like to submit a PR to the fedoraproject docs, but I kinda don't know what they're talking about and didn't really try to figure it out... I know how to "build from [raw] source" so I figured I'd make that work first, then think about updating the Fedora docs.

I've chosen to use a docker build environment.  This is ENTIRELY to provide a type of "cleanroom" environment and one of, what I think, is the perfect use case for `docker`.  It is in no way an endorsement of Docker for use in production...  I really kinda want to do it in a `chroot` but was being lazy.  This is BDD (Blog Driven Development) afterall!

Uhh... O3 is your choice...  if you don't know what that means, replace `-O3` below with `-O2`.  I made a bunch of changes and wasn't really scientific about it...

```
mkdir -p ~/dev/kernel/build && cd "$_"
cp /boot/config-`uname -r` ~/dev/kernel/system.config
git clone -b linux-5.2.y git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
docker run -it --name kernel-build --mount type=bind,source="${HOME}/dev/kernel",target=/kernel fedora:30
dnf install fedpkg fedora-packager rpmdevtools ncurses-devel pesign qt3-devel libXi-devel gcc-c++ flex bison elfutils-libelf-devel openssl-devel bc
cd /kernel/build
make olddefconfig
make menuconfig
KCFLAGS='-O3 -mtune=native -pipe' KCPPFLAGS='-O3 -mtune=native -pipe' make -j$(nproc) rpm-pkg LOCALVERSION="_$(date +%s)"
```

Uhh... you really shouldn't use this... but it's provided as refence...  I'd like to get into the specifics of what I did but haven't yet...

Gist of Current Kernel config: https://gist.github.com/qubitrenegade/c93e656a6fe1dd4f2e01aead3bebd2b4

* max clock speed (1000HZ) (actually seems to have been default?)
* make kernel "preemptable" (`SMP PREEMPT`)
* "disable 'NZ'" and legacy NZ (I forget the specific option)
* disable bunch of Intel options (though, not all?), since we're on an AMD proc...

Going to try the following:

```
perl -pi -e 's/^(CONFIG(:?.*)?.*(DEBUG|INTEL)(:?.*)?)=(.)/\1=n/' .config
make olddefconfig
KCFLAGS='-O3 -mtune=native -march=native -pipe' KCPPFLAGS='-O3 -mtune=cpu-type -march=cpu-type -pipe' make -j$(nproc) rpm-pkg
```

error

```
KCFLAGS='-O3 -march=native -pipe' KCPPFLAGS='-O3 5~-march=cpu-type -pipe' make -j$(nproc) rpm-pkg
```

Copy pasing helps! lol

```
KCFLAGS='-O3 -mtune=native -march=native -pipe' KCPPFLAGS='-O3 -mtune=native -march=native -pipe' make -j$(nproc) rpm-pkg
```


```
# Add name to make install easier

3c3
< # Linux/x86 5.2.8 Kernel Configuration
---
> # Linux/x86_64 5.2.7-200.fc30.x86_64 Kernel Configuration
23,24c23,24
< CONFIG_LOCALVERSION="-local"
< CONFIG_LOCALVERSION_AUTO=y
```

TODO:

* minimize kernel size...  atleast for now, hardware isn't going to change... maybe add another drive...


docker start kernel-build
docker exec -it kernel-build bash
docker stop kernel-build

THIS IS STILL A DRAFT!!! So ya.



