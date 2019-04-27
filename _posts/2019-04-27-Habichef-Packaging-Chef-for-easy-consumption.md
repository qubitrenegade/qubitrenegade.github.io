---
layout: post
title:  "HabiChef: Packaging Chef for easy consumption"
date:   2019-04-07 21:29:47 -0700
categories: Habitat Consul Docker
---

Dependencies are hard.  There are many different strategies to attempt to solve this problem.

## Policyfiles

Chef Environments have issues.  They're unversioned meaning they require an external process of managing version.  Once your environments get to a certain size, it can be a daunting task to update them.  Unless you are explicitly pinning the cookbook versions of every cookbook in your environment, you also expose yourself to risk when pushing changes to cookbooks.  In an environment where there may exist multiple Chef Servers that aren't all managed by the same team/process, communicating the updated version pinnings can also be a pain point.

Policy files are effectively an attempt to version environments and cookbook version pinnings while providing a simplified user interface.  This promotes safer workflows as new cookbooks published to the Chef Server won't be immediately consumed by all nodes, similar to managing version pinnings via environment.

## Habitat

Habitat automates applications by bundling the application--typically compiling from source, configuration, and logic to manage the configuration--including configuration updates--together.  However, not all applications lend themselves to being neatly packaged.  We all have those monolithic, CotS, prepackaged from the vendor, applications.

For those cases where our application can't be built from source, or doesn't lend itself to easily being packaged up: enter HabiChef.  With HabiChef we're bundling our standard set of Chef tools, which is the desired state of the system, in an easily distributable and manageable artifact.

## Example

Let's try an example.

Currently, there isn't a generator, so we're going to go at it by hand.  There's a skeleton project linked later on that you'll be able to clone and use.

### Skeleton File Structure

First we'll create our skeleton file structure.  Minimally we'll need a directory for our: project, policy files, and habitat plan.  In this example we'll also write our own cookbook, but the cookbook directory is optional.

```bash
$ mkdir -p habichef_demo/{cookbooks,habitat,policyfiles}
$ printf 'results\n.kitchen\n' > .gitignore
```

### Habitat Plan

Next we'll create the Habitat `plan.sh` and `default.toml` files.  We're going to assume we're not reporting to Automate for the purposes of this demo, refer to the project linked at the end for those configuration options.

#### `plan.sh`

The `plan.sh` is the brains of the operation.  This describes how to package and run our application.

```bash
# habichef_demo/habitat/plan.sh
scaffold_policy_name="base"
pkg_name="habichef_demo"
pkg_origin="qbrd"
pkg_version="0.1.0"
pkg_maintainer="QubitRenegade"
pkg_description="The Chef $scaffold_policy_name Policy"
pkg_upstream_url="http://chef.io"
pkg_scaffolding="core/scaffolding-chef"
pkg_svc_user="root"
pkg_svc_group="root"
```

We've discussed Habitat plans in the past so we won't delve too much into detail here.  What's interesting to note is that we've eschewed any of our standard hooks for leveraging this `core/scaffolding-chef` scaffolding.  This manages the `chef-client` in the same way as any other Habitat managed service (hooks etc.).

#### `defaut.toml`

The `default.toml` sets up default options for our Chef Client as a Service.  We need to set a few options here otherwise our scaffolding will fail with some less than obvious errors.

```toml
# habichef_demo/habitat/default.toml
log_level = "warn"
# All times are in seconds
interval = "1800"
splay = "300"
splay_first_run = "10"
run_lock_timeout = "30"

[data_collector]
  enable = false
```

These are fairly standard chef-client options.  Refer to the [`chef-client`](https://docs.chef.io/ctl_chef_client.html) documentation for full explanation of the remaining options.

### Policyfile

Next we'll create a basic policy file.  We'll modify this later, but for now, we want to include a cookbook from the Supermarket.

```ruby
# policyfiles/base.rb
name 'base'
# Where to find external cookbooks:
default_source :supermarket

# Environment Attributes
default['resolver']['nameservers'] = ['1.1.1.1']

# Your Run List
run_list [
  'resolver::default',
]
```

## Building Our Package

Now it's time to build our project!

If you haven't already, change into your habichef_demo directory. i.e.: assuming it's in your homedir:

```bash
$ cd ~/habichef_demo
```

If you're building from a CI tool, you likely want to build your package directly with `hab pkg build .` or similar.

### Habitat Studio

However, Habitat comes with a handy chroot environemnt.  We can enter this cleanroom build environment, then build and test the package there.

Let's enter the studio with `hab studio enter`, you'll see a lot of output, and it should drop you into a shell.

```
$ hab studio enter
[sudo hab-studio] password for qubitrenegade:
   hab-studio: Creating Studio at /hab/studios/home--qubitrenegade--habichef_demo (default)

... snip ...

   hab-studio: Entering Studio at /hab/studios/home--qubitrenegade--habichef_demo (default)
   hab-studio: Exported: HAB_ORIGIN=qbrd

--> Launching the Habitat Supervisor in the background...
    Running: hab sup run
    * Use 'hab svc start' & 'hab svc stop' to start and stop services
    * Use 'sup-log' to tail the Supervisor's output (Ctrl+c to stop)
    * Use 'sup-term' to terminate the Supervisor
    * To pass custom arguments to run the Supervisor, export
      'HAB_STUDIO_SUP' with the arguments before running
      'hab studio enter'.

--> To prevent a Supervisor from running automatically in your
    Studio, export 'HAB_STUDIO_SUP=false' before running
    'hab studio enter'.

[1][default:/src:0]#
```

### Build our Package

As we said above, from outside of the studio we can execute `hab pkg build .` to build our package.  Without any flags this will destroy any existing studios, spawn a new one, and build our package in the new studio.  However, since we are now in the Habitat Studio and therefore in a `chroot` environment, we can't run `hab pkg build .` as this will attempt to spawn a `chroot` jail inside a `chroot` jail.

From within our studio we need to issue the `build` command.

```
[1][default:/src:0]# build
   : Loading /src/habitat/plan.sh
   habichef_demo: Plan loaded
   habichef_demo: Validating plan metadata
   habichef_demo: Using HAB_BIN=/hab/pkgs/core/hab/0.79.1/20190410220617/bin/hab for installs, signing, and hashing
   habichef_demo: hab-plan-build setup
   habichef_demo: Writing pre_build file
   habichef_demo: Resolving scaffolding dependencies
» Installing core/scaffolding-chef

... snip ...

   habichef_demo: Installing /src/habitat/../policyfiles/base.rb
Building policy base
Expanded run list: recipe[resolver::default]
Caching Cookbooks...
Installing resolver 2.1.0

Lockfile written to /src/policyfiles/base.lock.json
Policy revision id: eb77ea2f4542b34427531c139129bd1bc026c57b12733c53442e2af1f5340293

... snip ...

   habichef_demo: hab-plan-build cleanup
   habichef_demo:
   habichef_demo: Source Path: /src
   habichef_demo: Installed Path: /hab/pkgs/qbrd/habichef_demo/0.1.0/20190428033409
   habichef_demo: Artifact: /src/results/qbrd-habichef_demo-0.1.0-20190428033409-x86_64-linux.hart
   habichef_demo: Build Report: /src/results/last_build.env
   habichef_demo: SHA256 Checksum: 1cacfcb1f90f9a162331fb85376a16e578bc9a6740e1eecbd3d2e78639fea111
   habichef_demo: Blake2b Checksum: 5e35d5950dc75e35ea7eb43f3a4e8745c6ac3c4cf350e51221bdbc0b546632b8
   habichef_demo:
   habichef_demo: I love it when a plan.sh comes together.
   habichef_demo:
   habichef_demo: Build time: 0m51s
[2][default:/src:0]#
```

Congrats!  You now have your first Habitat bundled Chef cookbook!

If we `ls` the `results/` directory, you should see a `.hart` artifact and a `last_build.env` file.

The `last_build.env` file contains information about the last build.

```
pkg_origin=qbrd
pkg_name=habichef_demo
pkg_version=0.1.0
pkg_release=20190428033409
pkg_target=x86_64-linux
pkg_ident=qbrd/habichef_demo/0.1.0/20190428033409
pkg_artifact=qbrd-habichef_demo-0.1.0-20190428033409-x86_64-linux.hart
pkg_sha256sum=1cacfcb1f90f9a162331fb85376a16e578bc9a6740e1eecbd3d2e78639fea111
pkg_blake2bsum=5e35d5950dc75e35ea7eb43f3a4e8745c6ac3c4cf350e51221bdbc0b546632b8
```

This means that we can source the file to export these variables to our environment.

Next, let's run it!

### Running our package in Habitat Studio

To make things easier, first, let's source our `last_build.env` file.

```
[4][default:/src:0]# source results/last_build.env
[5][default:/src:0]# echo $pkg_ident
qbrd/habichef_demo/0.1.0/20190428031740
```

Now that we have our environment variables loaded, we can run our package.

```
[6][default:/src:0]# hab svc load $pkg_ident
The qbrd/habichef_demo/0.1.0/20190428033409 service was successfully loaded
```

And then we'll start our package:

```
[7][default:/src:0]# hab svc start $pkg_ident
Supervisor starting qbrd/habichef_demo/0.1.0/20190428033409. See the Supervisor output for more details.
```

We can see our service is running with `hab svc status`

```
[8][default:/src:0]# hab svc status
package                                  type        desired  state  elapsed (s)  pid   group
qbrd/habichef_demo/0.1.0/20190428033409  standalone  up       up     2            1918  habichef_demo.default
```

If we view our log, with either `sup-log` or `less -R /hab/sup/default/sup.log`, we should see the update to the `resolv.conf`.


```
[9][default:/src:0]# sup-log
--> Tailing the Habitat Supervisor's output (use 'Ctrl+c' to stop)
habichef_demo.default(SR): Initializing
habichef_demo.default(SV): Starting service as user=root, group=root
habichef_demo.default(O): Starting Chef Client, version 14.12.9
habichef_demo.default(O): Using policy 'base' at revision 'eb77ea2f4542b34427531c139129bd1bc026c57b12733c53442e2af1f5340293'
habichef_demo.default(O): resolving cookbooks for run list: ["resolver::default@2.1.0 (6bbbd0f)"]
habichef_demo.default(O): Synchronizing Cookbooks:
habichef_demo.default(O):   - resolver (2.1.0)
habichef_demo.default(O): Installing Cookbook Gems:
habichef_demo.default(O): Compiling Cookbooks...
habichef_demo.default(O): Converging 1 resources
habichef_demo.default(O): Recipe: resolver::default
habichef_demo.default(O):   * template[/etc/resolv.conf] action create
habichef_demo.default(O):     - update content in file /etc/resolv.conf from 14236b to 82a27b
habichef_demo.default(O):     --- /etc/resolv.conf      2019-04-28 03:41:22.037991908 +0000
habichef_demo.default(O):     +++ /etc/.chef-resolv20190428-1929-1o2aqzr.conf   2019-04-28 03:41:27.218028370 +0000
habichef_demo.default(O):     @@ -1,5 +1,7 @@
habichef_demo.default(O):     -# Generated by NetworkManager
habichef_demo.default(O):     -nameserver 1.0.0.1
habichef_demo.default(O):     -nameserver 8.8.8.8
habichef_demo.default(O):     +#
habichef_demo.default(O):     +# This file is generated by Chef
habichef_demo.default(O):     +# Do not edit, changes will be overwritten
habichef_demo.default(O):     +#
habichef_demo.default(O):     +search
habichef_demo.default(O):     +nameserver 1.1.1.1
habichef_demo.default(O):
habichef_demo.default(O): Running handlers:
habichef_demo.default(O): Running handlers complete
habichef_demo.default(O): Chef Client finished, 1/1 resources updated in 01 seconds
```

## Conclusion

Today we've seen how we can bundle our Chef cookbooks using policy files to set the environment and run list.  The [habichef-skeleton](https://github.com/qubitrenegade/habichef-skeleton) repo contains the basic skeleton we've outlined here.  The [habichef_demo](https://github.com/qubitrenegade/habichef_demo) contains the project we've built here, there is a 2019-04-27 branch and tag.  You can also download and run the [Habitat package](https://bldr.habitat.sh/#/pkgs/qbrd/habichef_demo/latest) with a `hab pkg install qbrd/habichef_demo`.  

Tune in next time when we demonstrate adding local cookbooks and applying changes to running environments.
