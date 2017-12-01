---
layout: post
title:  "Managing Atomic hosts with Foreman and containerized Puppet"
date:   2017-12-01 17:00:00
categories: blog
---

So in the long silence since my last post I've been hard at work on my home-cluster
project. It's come quite a way in the last few months, and I've even been picking
up a few contributors along the way. At this point I can pretty reliably get a system
up that allows you to network boot a host, have it automatically provision with
Fedora Atomic, and then run the openshift-ansible installer against the hosts that Foreman
has detected, all in a single playbook. I also have created a [vagrant environment](https://github.com/fabianvf/home-cluster/tree/master/vagrant)
that makes it possible to create a virtual cluster with an arbitrary number of nodes
(limited of course by the power of your host).

While progress has been mostly smooth, I just wanted to highlight one technical issue
that I found no blog posts or documentation for (skip to the very end for a solution).

Pretty much immediately after my last post I realized that a good host management system was completely necessary, as
handling automatic provisioning with pxebooting is just horrible, and
having to manually provision things just defeats the purpose of this project.
I decided to use Foreman, because it had all the features I needed, and I have a lot of experience with it
(my first project at Red Hat was a Foreman plugin that handled automated provisioning
of RHV, OpenStack and OpenShift). There were a lot of fun problems, like creating a custom
Fedora Atomic compose that Foreman could serve when provisioning hosts,
creating and linking all of the resources required for provisioning through the Foreman
API, learning about DNS, certificates, content syncing, etc. I won't go into these issues,
because I was able to find fantastic resources for working my way through them, both in
blog posts and in the Foreman IRC channel.

Back to the issue that I hit for which I was unable to find any documentation for or record of whatsoever.

The basic problem was this:

1. When my nodes would PXEboot, their IP addresses would increment by one when Foreman assigned
    them their new hostname (if anyone can explain why this happens, it would be much appreciated).
2. Foreman had no way of knowing that their IP addresses changed, and since Foreman is in charge of DNS,
    my nodes were unreachable by name.

Foreman has tight integration with puppet, so my first thought was to run puppet on
the nodes, so they would report their updated host facts to Foreman, the DNS would
be updated, and everything would be awesome. It turns out this is a harder problem
than you'd expect, because in Fedora Atomic (and similar operating systems), most
of the file system is read only, and there is no way to install packages.
These operating systems pretty much exist so that you can run containers on them.
Technically, you can get packages into the operating system, by updating the definition for
the compose and creating a new build of the OS that includes the new packages. I didn't
want to do this because I have no desire to maintain my own compose (I currently
am creating a compose, but it's just to mirror content, it's very different from
maintaining a fork). Luckily, containerized puppet actually does exist. All I needed to do
was get a puppet configuration file and set the docker container to run automatically
on startup. Normally, in the kickstart you would just tell the puppet service to start,
but you can't do that with containers. This is pretty much how two weeks of my free time went:

"It's an easy enough workaround, you just need to docker pull and run the container with a 'restart always'
policy when you're configuring the system."

"Weird, I don't see any containers pulled down, let me check out the anaconda kickstart log..."

"Oh, right, everything is chrooted during anaconda install so you can't actually start the docker process."

"Guess I'll just have to make a cron job to run it on startup then."

"Oh neat, cron isn't included on Atomic systems by default."

"I guess I can just define my own systemd service that will handle bringing the container up and down."

This one actually probably would have worked, but I'm an idiot and can't set up a systemd service.

At this point I was pretty stumped. I needed the container to be configured to run before
the host was available for any provisioning outside of the kickstart, but as far as I could
tell I had no way to do that. Then, as I was scanning the kickstart for the nth time, I realized
that cloud-init services were being disabled. Then I went to get a coffee, and about two hours
after that a lightbulb went off. If the cloud-init services were being disabled, that meant I could
have access to cloud-init if I just enabled them. I made a slight modification to the kickstart to
configure a cloud-init job that would run on startup, totally borked the kickstart about 6 times
before I figured out how to escape a semi-colon in an argument to the bootloader, and reran my deployment.

It worked!

Kind of.

Foreman was indeed getting puppet reports from the nodes, and the host entries *in Foreman* had the
updated IP addresses. But DNS was still broken, even though Foreman was updated, if I did a lookup
of the hostname it was returning the old IP. At this point I was starting to get a little frustrated, but
I asked around on IRC and poked around in the code, and found out that Foreman explicitly disables
updating things like DNS when importing facts from a host. There's a button and API that allow you to
manually rebuild the configuration option, but that's it. Luckily, the Foreman community is great,
so I filed an [issue](http://projects.theforeman.org/issues/21565), followed up with a
[pull request](https://github.com/theforeman/foreman/pull/4976), went through a little bit of code review, and
as a result Foreman 1.17.0 will have support for a configuration option that will update the DNS
when puppet reports an IP address change.

Now, for the patient readers who are by this point probably regretting the minutes they spent on my rambles,
and the impatient readers who probably make up the overwhelming majority of my audience, here's a summary of the kickstart
modifications that were required to get containerized puppet configured to run during a kickstart install of Fedora Atomic:

    <snip>
    bootloader --timeout=3 --append="ds=nocloud\;seedfrom=/var/cloud-init/"
    services --enabled cloud-init,cloud-config,cloud-final,cloud-init-local,sshd,docker

    %post

    mkdir -p /etc/puppetlabs/puppet
    # This snippet is provided by Foreman, and requires no modification to work in a containerized environment
    cat <<EOF >/etc/puppetlabs/puppet/puppet.conf
    <%= snippet 'puppet.conf' %>
    EOF

    #------------ configure cloud-init to run containerized puppet --------------
    mkdir /var/cloud-init

    cat << EOF>/var/cloud-init/meta-data
    instance-id: iid-local<%= @host.id %>
    local-hostname: <%= @host %>
    EOF

    cat << EOF>/var/cloud-init/user-data
    #cloud-config
    runcmd:
    - "systemctl start docker"
    - "docker pull docker.io/puppet/puppet-agent"
    - "docker run -d --restart always --name puppet-agent --hostname <%= @host %> --net host -v /tmp:/tmp -v /etc:/etc -v /var:/var -v /usr:/usr -v /lib64:/lib64 --privileged puppet/puppet-agent agent --no-daemonize --certname <%= @host.certname %> --server <%= @host.puppetmaster %>"
    EOF

    </snip>

Cheers,
\- Fabian
