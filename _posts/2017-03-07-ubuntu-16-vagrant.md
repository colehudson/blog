---
layout: post
title: Developing with Vagrant and Ubuntu 16.04
summary: Changing over to Ubuntu 16.04 in Vagrant isn't so easy
categories: vagrant virtualbox ubuntu 16.04
published: true
---
## Developing with Vagrant and Ubuntu 16.04

When upgrading my Vagrant-run development environment to Ubuntu 16.04 (xenial) from 14.04 (trusty), I encountered a few vexing issues. Here's what I did to fix them.

1) Make sure your versions of Vagrant and Virtualbox (if this is your provider) are up-to-date.

2) The normal Ubuntu Vagrant box does not seem to work. In other words, changing your box to config.vm.box = "ubuntu/xenial64" is going to be open a world of hurt. See [here](https://bugs.launchpad.net/cloud-images/+bug/1569237) and [here](https://github.com/mitchellh/vagrant/issues/7155#issuecomment-228568200) for more. I've found success with the boxes made by Bento.
Solution:
Change Vagrant Box to
{% highlight ruby lineanchors %}
config.vm.box = "bento/ubuntu-16.04"
{% endhighlight %}

3) With Ubuntu 16.04, there seems to be a recurring issue with running upgrades. With the version of I am using at the time of this post (bento/ubuntu-16.04 v2.3.1), it can trigger an upgrade screen for Grub. This breaks the ability to run a provisioning script non-interactively. The issue seems to pop-up occasionally; see [here](https://github.com/chef/bento/issues/661).

Solution:
[Tell the system to handle any prompts non-interactively](http://stackoverflow.com/questions/40748363/virtual-machine-apt-get-grub-issue/40751712)
{% highlight bash lineanchors %}
export DEBIAN_FRONTEND=noninteractive apt-get -y -o DPkg::options::="--force-confdef" -o DPkg::options::="--force-confold" upgrade
{% endhighlight %}
