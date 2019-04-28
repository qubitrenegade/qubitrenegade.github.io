---
layout: post
title:  "HabiChef: Pt2 Applying Change"
date:   2019-04-28 15:29:47 -0700
categories: Habitat Chef HabiChef
---

In our [last post](https://qubitrenegade.com/habitat/chef/habichef/2019/04/28/Habichef-Packaging-Chef-for-easy-consumption.html) we discussed how to package our Chef Cookbooks in a Habitat package using Policyfiles.  We showed that we could package all of our cookbooks into a single, easily distributable, package.

We were even able to set node attributes via the Policyfile enabling us to create Policyfile specific settings.  However, we don't want to manage a dozen different versions of the resolver package since the only thing that changes is node attributes.

For instance, all of our servers use the same DNS server but will have a different domain based on the data center they live in.  We'll use `qubitrenegade.com`, so suppose we have `us-east-1.qubitrenegade.com` and `us-west-1.qubitreengade.com`.

## Setup

First, let's update our base policy file.  Let's add a `domain` attribute to our `default` settings.

```
--- a/policyfiles/base.rb
+++ b/policyfiles/base.rb
@@ -4,7 +4,10 @@ name 'base'
 default_source :supermarket

 # Environment Attributes
-default['resolver']['nameservers'] = ['1.1.1.1']
+default['resolver'] = {
+  'domain' = 'qubitrenegade.com',
+  'nameservers' = ['1.1.1.1'],
+}
```

## Multiple Policyfiles

Since policy files include other policy files, we could create a new policy for each environment.

First, let's update our Habitat `plan.sh` to include the policy name in the name of our package.

```
--- a/habitat/plan.sh
+++ b/habitat/plan.sh
@@ -1,6 +1,11 @@
 # habichef_demo/habitat/plan.sh
-scaffold_policy_name="base"
-pkg_name="habichef_demo"
+if [ -z ${CHEF_POLICYFILE+x} ]; then
+  policy_name="base"
+else
+  policy_name="${CHEF_POLICYFILE}"
+fi
+
+pkg_name="habichef_demo-${policy_name}"
```

Next, let's create a Policyfile for `us-east-1` that includes our base Policyfile.

```
# habichef_demo/policyfiles/us-east-1.rb
name 'us-east-1'

include_policy 'base'

default['resolver']['domain'] = 'us-east-1.qubitrenegade.com'
```

Then we can build our project, using the same process as before.  (If you're receiving errors, you can check out the `2019-04-28-pt1-multiple-policy-files` branch of the [HabiChef Demo repo](https://github.com/qubitrenegade/habichef_demo/tree/2019-04-28-pt1-multiple-policy-files).

```bash
$ hab studio enter

... snip ...

[1][default:/src:0]# build
... snip ...

  habichef_demo-base: Installed Path: /hab/pkgs/qbrd/habichef_demo-base/0.1.0/20190428185221
   habichef_demo-base: Artifact: /src/results/qbrd-habichef_demo-base-0.1.0-20190428185221-x86_64-linux.hart
   habichef_demo-base: Build Report: /src/results/last_build.env
   habichef_demo-base: SHA256 Checksum: 4a126ec5c47dba1643ef767c7ccc02acd36ea780d2f88110db9c964c4598c2b9
   habichef_demo-base: Blake2b Checksum: 7fc45fcf07bebe57f90301c0c83df6700f772fc8925ec0162b1efbfa85b89b6a
   habichef_demo-base:
   habichef_demo-base: I love it when a plan.sh comes together.
   habichef_demo-base:
   habichef_demo-base: Build time: 0m5s
[2][default:/src:0]#
```

However, note that the name of our package is `habichef_demo-base`.  We need to set `
CHEF_POLICYFILE` environment variable to build the correct package.

```
[2][default:/src:0]# CHEF_POLICYFILE=us-east-1 build
... snip ...
Building policy us-east-1
Error: Failed to generate Policyfile.lock
Reason: (ChefDK::LocalPolicyfileLockNotFound) The provided path ./base.lock.json does not exist.
```

Well, it seems that something is blowing away the `base.lock.json` file...  This probably needs follow up with the Habitat team.

Regardless, this example starts to break down as you start to deploy to more environments as you have effectively the same package being generated with minor changes.

## `user.toml` Config Changes

Let's continue working with `habichef_demo` package.

If you've been following along, we'll want to explicitly set our `scaffold_policy_name="base"`

```
--- a/habitat/plan.sh
+++ b/habitat/plan.sh
@@ -5,8 +5,8 @@ else
   policy_name="${CHEF_POLICYFILE}"
 fi

-scaffold_policy_name="${policy_name}"
-pkg_name="habichef_demo-${policy_name}"
+scaffold_policy_name="base"
+pkg_name="habichef_demo"
```

(You can also remove the conditional statement, or just check out the `2019-04-28-pt2-user.toml` branch)

Next, let's create our `user.toml` file.  Habitat will load user config settings from `/hab/user/<service name>/config/user.toml`.

```
[15][default:/src:130]# mkdir -p /hab/user/habichef_demo/config/
[16][default:/src:0]# printf '[attributes]\n  [attributes.resolver]\n    domain = "us-east-1.qubitrenegade.com"\n' | tee /hab/user/habichef_demo/config/user.toml
[attributes]
  [attributes.resolver]
    domain = "us-east-1.qubitrenegade.com"
```

Then we'll build and run the package.

```
$ hab studio enter
... snip ...
[1][default:/src:0]# build
... snip ...
   habichef_demo: Installed Path: /hab/pkgs/qbrd/habichef_demo/0.1.0/20190428192425
   habichef_demo: Artifact: /src/results/qbrd-habichef_demo-0.1.0-20190428192425-x86_64-linux.hart
   habichef_demo: Build Report: /src/results/last_build.env
   habichef_demo: SHA256 Checksum: b0016e36ae76f77e876e27681219eae415eb84ef5801813d83fbd56c46163099
   habichef_demo: Blake2b Checksum: 1be21e1176536917f28b50f3b2856acdc7ad51116009c8aee23ef7bb48708880
   habichef_demo:
   habichef_demo: I love it when a plan.sh comes together.
   habichef_demo:
   habichef_demo: Build time: 0m7s
[2][default:/src:0]# source results/last_build.env
[3][default:/src:0]# hab svc load $pkg_ident
[4][default:/src:0]# sup-log
... snip ...
habichef_demo.default(O):     +# This file is generated by Chef
habichef_demo.default(O):     +# Do not edit, changes will be overwritten
habichef_demo.default(O):     +#
habichef_demo.default(O):     +domain us-east-1.qubitrenegade.com
habichef_demo.default(O):     +nameserver 1.1.1.1
```

As you can see, you can set per-instance settings via `user.toml`.  Typicaly this is populated by a third-party tool such as Chef (though in the case of HabiChef, is it the chicken or the egg?), or Terraform.  Terraform is particularly useful when coupled with the Habitat provisioner.

## Running config changes

We've been running HabiChef for a few weeks now and have several nodes deployed (possibly even with a bastion ring, but that is a topic for another day). A mandate comes down from management that we need to change all of the domains.  Sure, we could log into every node and update the `user.toml` file... but that seems like a lot of work and rather error prone.

Fortunately, Habitat provides a method for applying config changes to a running cluster.

First, let's create a temporary `.toml` file.  (There is an example file in the `habichef_demo/habitat` directory)

```
# /tmp/config-demo.toml
[attributes]
  [attributes.resolver]
    domain = "example.local"
```

Then we can apply it with:

```
hab config apply <SERVICE_GROUP> <VERSION_NUMBER> <FILE>
```

We'll use a timestamp generated with `$(date +%s)` as the version number.  This way when we apply the update again we don't have to modify the command.

Our command should look something like:

```
hab config apply habichef_demo.default $(date +%s) habitat/config-apply-demo.toml
```

And we should see something to the effect of:

```
[2][default:/src:0]# hab config apply habichef_demo.default $(date +%s) habitat/config-apply-demo.toml && sup-log

... snip ...

hab-sup(CMD): Setting new configuration version 1556489698 for habichef_demo.default
habichef_demo.default(CF): Modified configuration file /hab/svc/habichef_demo/config/attributes.json

... snip ...

habichef_demo.default(O):   * template[/etc/resolv.conf] action create
habichef_demo.default(O):     - update content in file /etc/resolv.conf from 217c99 to 19c14a
habichef_demo.default(O):     --- /etc/resolv.conf      2019-04-28 19:44:54.719939924 +0000
habichef_demo.default(O):     +++ /etc/.chef-resolv20190428-32280-15wxpjo.conf  2019-04-28 22:15:11.311781489 +0000
habichef_demo.default(O):     @@ -2,7 +2,7 @@
habichef_demo.default(O):      # This file is generated by Chef
habichef_demo.default(O):      # Do not edit, changes will be overwritten
habichef_demo.default(O):      #
habichef_demo.default(O):     -domain asdf.com
habichef_demo.default(O):     +domain example.local
```

Success!  Typically these changes would be applied to a cluster, and the update propagated through the cluster via Habitat's inbuilt gossip layer.  So if you're thinking this is a lot of work to change the domain in the `resolv.conf` file, you're not wrong.

## Conclusion

Today we've seen three different ways (well, two and a half) to apply changes to HabiChef cookbooks.  Big picture, you'd probably want to manage these via some form of CI pipeline, and discourage users from manually `hab config apply`ing changes.  Hopefully the gravity of being able to apply config changes to a cluster will become more apparent in the "Habitat: PtX Running at Scale".

Tune in next time when we discuss the internals of `scaffolding-chef` and how to include local cookbooks.
