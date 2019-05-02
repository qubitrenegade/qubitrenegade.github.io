---
layout: post
title:  "HabiChef: Pt3 HabiChef Internals"
date:   2019-05-08 22:08:42 -0700
categories: Habitat Chef HabiChef
---

So far we've talked about how to bundle our Chef cookbooks using Policyfiles and shown how to apply changes.

Meanwhile back at the farm... we've received mandate to change our DNS servers from `1.1.1.1` to `1.0.0.1`.  Since we have a running cookbook that manages the `resolv.conf` file, this should be easy.

## Should is a funny word

First, let's enter our studio, and install and load our service.  If our service is already installed or loaded, well get a message as such.

```
$ hab studio enter
[1][default:/src:0]# source results/last_build.env
[2][default:/src:0]# hab pkg install "results/${pkg_artifact}"
» Installing results/qbrd-habichef_demo-0.1.0-20190428193627-x86_64-linux.hart
→ Using qbrd/habichef_demo/0.1.0/20190428193627
★ Install of qbrd/habichef_demo/0.1.0/20190428193627 complete with 0 new packages installed.
[4][default:/src:0]# hab svc load $pkg_ident
✗✗✗
✗✗✗ [Err: 3] Service already loaded, unload 'qbrd/habichef_demo/0.1.0/20190428193627' and try again
✗✗✗
```

Next, let's create our config file.  Again this can be found in the Habitat directory.

```
[5][default:/src:1]# cat habitat/config-apply-demo.toml
# habichef_demo/habitat/config-apply-demo.toml

[attributes]
  [attributes.resolver]
    # domain = "example.local"
    nameservers = ["1.0.0.1"]
```

And apply our config.

```
[7][default:/src:0]# hab config apply habichef_demo.default $(date +%s) habitat/config-apply-demo.toml && sup-log

... snip ...

hab-sup(CMD): Setting new configuration version 1556494201 for habichef_demo.default
habichef_demo.default(CF): Modified configuration file /hab/svc/habichef_demo/config/attributes.json

habichef_demo.default(O):     -nameserver 1.1.1.1
habichef_demo.default(O):     +nameserver 1.0.0.1
```

Great, but no sooner have we made this change does the decision come down that now we need to update to use only `8.8.8.8` dns server.

Ok, we're getting good at this now. Create our toml file, apply it, op success!

```
[10][default:/src:0]# cat habitat/config-apply-demo.toml
# habichef_demo/habitat/config-apply-demo.toml

[attributes]
  [attributes.resolver]
    nameservers = ["8.8.8.8"]
[12][default:/src:0]# hab config apply habichef_demo.default $(date +%s) habitat/config-apply-demo.toml && sup-log
» Setting new configuration version 1556494812 for habichef_demo.default

habichef_demo.default(O):   * template[/etc/resolv.conf] action create
habichef_demo.default(O):     - update content in file /etc/resolv.conf from 3a75c3 to b0922e
habichef_demo.default(O):     --- /etc/resolv.conf      2019-04-28 23:36:57.890929043 +0000
habichef_demo.default(O):     +++ /etc/.chef-resolv20190428-8076-x1jtob.conf    2019-04-28 23:40:25.293603966 +0000
habichef_demo.default(O):     @@ -5,4 +5,5 @@
habichef_demo.default(O):      domain qubitrenegade.com
habichef_demo.default(O):      nameserver 1.0.0.1
habichef_demo.default(O):     +nameserver 8.8.8.8
```

Wait, what?  That just added `8.8.8.8` as a secondary DNS server, but shouldn't it have replaced `1.0.0.1` with `8.8.8.8`?

## Chef, Attributes, and Arrays

As it turns out, no, not exactly.  Our attributes set through Habitat are being set as `normal` attributes.  [Noah does a great job](https://coderanger.net/arrays-and-chef/) explaining the finer points, the short version is `normal` attributes persist through `chef-client` runs and array attributes are merged.

Now we understand why this is happening, let's look at how.

## Habitat: Peeking Under the Hood

When we look at our Habitat `plan.sh` file, it's rather uninteresting and fairly standard boilerplate.  This really doesn't change from cookbook set to cookbook set, so all of the juicy bits have been abstracted into the Chef scaffolding.

Habitat packages consist of lifecycle hooks templates that describe how to run the service service and config file templates that set configuration settings.  These config files are then rendered into `/hab/svc/<service name>` where they are are consumed.

Let's look at our service run hook.  Once again in the studio:

```
[14][default:/src:0]# cat /hab/svc/habichef_demo/hooks/run
#!/bin/sh

chef_client_cmd()
{
  chef-client -z -l warn -c /hab/svc/habichef_demo/config/client-config.rb -j /hab/svc/habichef_demo/config/attributes.json --once --no-fork --run-lock-timeout 30
}

... snip ...

while true; do

sleep $SPLAY_DURATION
sleep 1800
chef_client_cmd
done
```

We're skiping some of the middle bits here, but the two important pieces are, we define a function `chef_client_cmd` that returns the command to run the chef client.  Then we run it in a loop (sleeping for 1800s, or 30mins).

What we see here is that our attributes are being passed in with the `-j` flag, which we can see are how we pass json attributes to the chef client.

```
[17][default:/src:0]# hab pkg exec chef/chef-client chef-client --help | grep j
    -j JSON_ATTRIBS,                 Load attributes from a JSON file or URL
        --json-attributes
```

And [the documentation](https://docs.chef.io/ctl_chef_client.html) warns us that:

> All attributes are normal attributes
>
> Any other attribute type that is contained in this JSON file will be treated as a normal attribute. Setting attributes at other precedence levels is not possible.

What this means is if we want to be able to pass attributes to our package at run time, we have to deal with the `normal` attributes.  In _most_ cases, this isn't an issue, as we demonstrated with changing our domains.  However, in the case where our attribute is an array, we're going to run into issues.  Do not despair, this is a solvable problem!

## Local Wrapper Cookbook

A common pattern in Chef is the [Wrapper Cookbook](https://blog.chef.io/2017/02/14/writing-wrapper-cookbooks/).  Jerry does a great job explaining the finer points, the nutshell is a Wrapper Cookbook is a Cookbook that includes other Cookbooks, often setting attributes.

In our case, we only want to include one other cookbook, but in a real life scenario we would probably want to include more than one other cookbook.

### Setup

First, let's generate our cookbook and `cd` into our cookbook dir.

```
$ cd ~/habichef_demo
$ chef generate cookbook cookbooks/my_resolver --copyright QubitRenegade -m 'q at qubitrenegade period com' --kitchen dokken --license mit
$ cd ~/habichef_demo/cookbooks/my_resolver
```

### Tests

Next, let's write our tests.  Tests are important to ensure we don't break things when we change them in the future.  The reason we write them first is it gives us a handy checklist for what we need our code to do.

We need to add the following two lines to our `spec/spec_helper.rb` to load our helper library and add coverage tests.

```
Dir['libraries/*.rb'].each { |f| require File.expand_path(f) }
at_exit { ChefSpec::Coverage.report! }
```

Then we need to update `spec/unit/recipes/default_spec.rb` and add the following line to each relevant context block.

```
it { is_expected.to include_recipe 'resolver::default' }
```

For our final test we'll create a helper, `spec/unit/libraries/` directory, and a `helpers_spec.rb` for our tests.

```
$ chef generate helpers helpers
$ mkdir -p spec/unit/libraries
$ cat spec/unit/libraries/helpers_spec.rb
require 'spec_helper'

describe MyResolver::RecipeHelpers do
  subject do
    Object.new.extend(described_class)
  end

  before do
    subject.define_singleton_method(:node) do
      {
        'my_resolver' => {
          'nameservers' => {
            '1.1.1.1' => true,
            '0.0.0.0' => false,
            '8.8.8.8' => true,
            '9.9.9.9' => false,
          }
        }
      }
    end
  end

  describe "#map_resolvers" do
    it "matches the nameservers that are `true'" do
      expect(subject.map_nameservers).to match_array ['1.1.1.1', '8.8.8.8']
    end
  end
end
```

What we're doing here is creating a fake class and assigning it to `subject`.  Then we create a method called `node` that returns our mocked out node attributes.

### Helper


```
module MyResolver
  module RecipeHelpers
    def map_nameservers
      node['my_resolver']['nameservers'].select{|k,v| v}.map{|k,v| k}
    end
  end
end
```

This may be a bit obtuse for the Ruby uninitiated.

A more verbose way of writing this could be:

```
def map_nameservers
      node['my_resolver']['nameservers']
        .select{|_ip ,include| include}
        .map{|ip ,_include| ip}
```

Another way of writing this could be:

```
def map_nameservers
  active_ns = []
  node['my_resolver']['nameservers'].each do |ip, include|
    active_ns.push ip if include
  end
  active_ns
end
```

`select` iterates through our map and returns a list of hashes where "condition" is true (`v` since our value is `true` or `false`).  Map then transforms our array by returning an array of `k`.

### Cookbook

```
extend MyResolver::RecipeHelpers
node.override['resolver']['nameservers'] = map_nameservers
include_recipe 'resolver::default'
```
