---
layout: post
title:  "Chef: Getting Started"
date:   2018-02-01 12:13:57 -0700
categories: Intro Chef
---

## Introduction

This does not intend to be a replacement for the tutorials on the [Chef Website](https://learn.chef.io/).  In fact, I higly recommend reading and working through those tutorials as this blog intends to build on ideas and concepts presented there.  In fact, most of this guide is based on the [Chef Tutorial](https://learn.chef.io/modules/manage-a-node-chef-server/ubuntu/bring-your-own-system/set-up-your-workstation#/) and [Documentation](https://docs.chef.io/workstation.html) on the same subject.

This page exists to document my workflows.  So without further ado.

## Chef Account Setup (Hosted Chef)

Head on over to the [Sign Up Page](https://manage.chef.io/signup) to get signed up with a Hosted Chef.  You can run Chef.io Hosted Chef for free up to 5 nodes.  If you need more than that then you're looking at paying for nodes, or hosting your own Chef Server.

### Knife Setup

Head over to [The Organizations page](https://manage.chef.io/organizations/) in your manage interface (Administration -> Organizations).  Find your organization, and select "Generate Knife Config".  You should download a {% ihighlight %}knife.rb{% endihighlight %} file.

Then head over to Users (Administration -> Users) and find your user, then select "Reset Key".  You should download a {% ihighlight %} <username>.pem {% endihighlight %} file.

Create a {% ihighlight %} ~/.chef {% endihighlight %} directory, and move your newly downloaded files into the {% ihighlight %} ~/.chef {% endihighlight %} directory.

{% highlight bash %}
$ mkdir ~/.chef
$ mv ~/Downloads/{knife.rb,qubitrenegade.pem} ~/.chef
{% endihighlight %}

### Knife.rb file

Your {% ihighlight %}knife.rb{% endihighlight %} will look like

{% highlight ruby %}
# See http://docs.chef.io/config_rb_knife.html for more information on knife configuration options

current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                "qubitrenegade"
client_key               "#{current_dir}/qubitrenegade.pem"
chef_server_url          "https://api.chef.io/organizations/qubitrenegade"
cookbook_path            ["#{current_dir}/../cookbooks"]
{% endhighlight %}

## ChefDK Install

This guide assumes you're not doing other Ruby development.  If you use bundler and things get weird (Chef can't find gems that are installed when you {%ihighlight%}gem list{% endihighlight %}), you can remove the {% ihighlight %}~/.chefdk/gems{% endihighlight %}  directory.

### Download ChefDK

Head over to the [ChefDK Downloads page}(https://downloads.chef.io/chefdk) and select the appropriate package.  For me, I selected the Red Hat 7 rpm.

{% highlight bash %}
$ wget -P /tmp https://packages.chef.io/files/stable/chefdk/2.4.17/el/7/chefdk-2.4.17-1.el7.x86_64.rpm
2018-02-04 17:34:40 (3.33 MB/s) - ‘/tmp/chefdk-2.4.17-1.el7.x86_64.rpm’ saved [106718859/106718859]

$ sha256sum /tmp/chefdk-2.4.17-1.el7.x86_64.rpm 
ab2d62b1eeb5248949f044f4c4bf489babe652fc649dc93cc2dac30607eb207e  /tmp/chefdk-2.4.17-1.el7.x86_64.rpm

$ sudo dnf install /tmp/chefdk-2.4.17-1.el7.x86_64.rpm
Running transaction
  Preparing        :                                                                                                                                                                                           1/1 
  Installing       : chefdk-2.4.17-1.el7.x86_64                                                                                                                                                                1/1 
  Running scriptlet: chefdk-2.4.17-1.el7.x86_64                                                                                                                                                                1/1 
Thank you for installing Chef Development Kit!
  Verifying        : chefdk-2.4.17-1.el7.x86_64                                                                                                                                                                1/1 

Installed:
  chefdk.x86_64 2.4.17-1.el7                                                                                                                                                                                       

Complete!
{% endhighlight %}

### Configure ChefDK

Once our ChefDK package is installed, we need to set up our shell. (Windows users, head over to [ChefDK on Windows](https://docs.chef.io/dk_windows.html#configure-environment)) 

{% highlight bash %}
echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile
{% endhighlight %}

Then we can either log in a new shell, or just source our file:

{% highlight bash %}
source ~/.bash_profile
{% endhighlight %}

If you're using rbenv to manage Ruby, you're own your own for now...

## Verify

At this point you should have the ChefDK installed and should be able to communicate with your Hosted Chef account.  We can verify by running a few commands:

{% highlight bash %}
$ chef -v
Chef Development Kit Version: 2.4.17
chef-client version: 13.6.4
delivery version: master (73ebb72a6c42b3d2ff5370c476be800fee7e5427)
berks version: 6.3.1
kitchen version: 1.19.2
inspec version: 1.45.13

$ knife client list
<your orgname>-validator
node1
node2
node3
{% endhighlight %}

(note: I already have nodes registered, you likely will not.  So you'll likely only see the validator key)

## SSL Errors

If you're working with self signed certs or are behind some corporate proxies, Ruby will have a bad time validating SSL certs.  While the _best_ answer is always "fix your SSL", but that is, unfortunately, not always a possible solution.  These can be worked around with the below solutions.

### Chef SSL Errors

#### Download SSL Certs

The first thing we can try is to download the SSL certs from the server.

{% highlight bash %}
$ knife ssl fetch
WARNING: Certificates from api.chef.io will be fetched and placed in your trusted_cert
directory (/home/qubitrenegade/.chef/trusted_certs).

Knife has no means to verify these are the correct certificates. You should
verify the authenticity of these certificates after downloading.

Adding certificate for wildcard_opscode_com in /home/qubitrenegade/.chef/trusted_certs/wildcard_opscode_com.crt
Adding certificate for DigiCert_SHA2_Secure_Server_CA in /home/qubitrenegade/.chef/trusted_certs/DigiCert_SHA2_Secure_Server_CA.crt
{% endhighlight %}

#### Disable SSL Checking

If you're dealing with a proxy, the above may not work still.  You can disable ssl checking with knife entirely, though this is not recommended, disabling SSL for Knife is documented below.

{% highlight ruby %}
echo 'ssl_verify_mode :verify_none' >> ~/.chef/knife.rb
{% endhighlight %}

Again, this disables all SSL checking and is not recommended!!!

### Berkshelf SSL Errors

Disable ssl checking for berkshelf:

{% highlight bash %}
mkdir ~/.berkshelf
echo '{"ssl":{"verify": false }}' > ~/.berkshelf/config.json
{% endhighlight %}

## Docker

This should not be seen as condoning the use of docker in production!  However, for the purposes of the next few posts, it will serve our needs.  Again, this post mostly just follows the instructions found in the [offical documentation](https://docs.docker.com/install/linux/docker-ce/fedora/)

{% highlight bash %}
# Remove old versions of Docker
sudo dnf -y remove docker docker-common docker-selinux docker-engine-selinux docker-engine

# install dnf plugins
sudo dnf -y install dnf-plugins-core

# add docker-ce repo
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo

# install docker-ce
sudo dnf -y install docker-ce 

# start docker
sudo systemctl start docker

# optionally enable docker to start at boot
sudo systemctl enable docker

# and validate we can run a container
sudo docker run hello-world

# if you get an error like the following, your daemon probably isn't started.
$ sudo docker run hello-world
docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?.
See 'docker run --help'.

# Add your user to the docker group.
# This doesn't seem to work on F27?  Further investigation to follow
# Maybe try logging out then logging back in.
$ sudo usermod -aG docker $USER
{% endhighlight %}

## First cookbook

I like to do my work in my {% ihighlight %}~/chef-dev{% endihighlight %} directory.  So let's create this first.

{% highlight bash %}
$ mkdir ~/chef-dev
$ cd ~/chef-dev
{% endhighlight %}

Then we'll create our first cookbook, this can be found on [github](https://github.com/qubitrenegade/my_test), but it's not much more than a base {% ihighlight %}chef generate cookbook <cookbook name>{% endihighlight %}.

{% highlight %}
$ chef generate cookbook my_test
$ cd my_test
{% endhighlight %}

### Running Test Kitchen

At this point, we should be in our new cookbook directory, and be able to run {% ihighlight %}kitchen test{% endihighlight %}, but I'm having trouble and didn't want to reboot at the moment...

So more on this later,

## Conclusion

At this point, we should have:

* installed the ChefDK, 
* installed Docker, 
* Configured our {% ihighlight %}knife.rb{% endihighlight %}
* created our first cookbook using the ChefDK generator
* Run Test Kitchen

We're now ready to rock!  Tune in next time when we discuss the coobkook development process.

- Q
