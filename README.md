#### Table of Contents

1. [Overview](#overview)
  * [Supported Software](#supported-software)
  * [Network](#network)
  * [Users](#users)
  * [Git](#git)
2. [Preparation](#preparation)
3. [Git Modifications](#git-modifications)
4. [Bootstrap](#bootstrap) - Create the puppet master
  * [Automated Install](#automated-install)
  * [Manual Install](#manual-install)
5. [Creating Your First Managed Node](#creating-your-first-managed-node) - Add a DNS server
6. [Additional Nodes](#additional-nodes) - Instructions for additional puppetized nodes
7. [Get Labbing!](#get-labbing)
8. [Roadmap](#roadmap)
9. [BUGS](#bugs)

## Overview

This project aims to provide the user with a puppetized lab setup. It can be
used as the foundation of a home network, a *learning puppet* environment, or
as a proof of concept lab for non-puppet software and hardware. Once complete,
the lab environment will provide a number of services to the designated
network. Each service definition includes a valid SELinux security configuration.

  * dns
  * dhcp
  * puppet
  * puppetdb
  * hiera
  * r10k
  * yum repository
  * build server
  * mysql
  * phpmyadmin
  * tftp

The user must provide a default gateway for external configuration (see
[network](#network) for more information).

Puppetinabox was covered on the 1/14/2015 vBrownBag DevOps Series. Watch 
the video and [check out the slides]
(http://www.slideshare.net/rnelson0/vbrownbag-devops-series-puppetinabox)!

### Supported Software
**OS**: CentOS 6.5 and 6.6 have been tested thoroughly, specifically with
[my CentOS kickstart](http://rnelson0.com/2014/04/18/kickstart-your-centos-template/).
Other Linux distributions may work. Please report any successes or failures via
issues. Unless otherwise stated, all nodes are assumed to be CentOS 6.5.

**Puppet**: Tested with Puppet 3.7.3. Your OS templates should all include
puppet.

### Network
The lab includes sample data and dns zone files describing 10.0.0.0/24. The
DNS suffix is *example.com*. The following IP assignments are suggested and
should be configured in your OS templates:

  * 10.0.0.1 - default gw (user provided)
  * 10.0.0.5 - puppet
  * 10.0.0.10 - build
  * 10.0.0.11 - phpmyadmin
  * 10.0.0.40 - mysql
  * 10.0.0.251 - tftp
  * 10.0.0.252 - yumrepo (CNAME yum)
  * 10.0.0.253 - dns
  * 10.0.0.254 - dhcp

You are encouraged to change the DNS suffix and IP assignments. The
documentation will make use of the sample suffix and IPs, however.

### Users

In your OS template, set a known root password of sufficient complexity for
your environment. During the initial puppet run, an additional non-privileged
user will be created:

  * padmin: Pupp3tl4b

After the user is created on the first puppet run, you should use *padmin*
rather than root.

An SSH key is included that can used with clients supporting the PuTTy (.ppk),
OpenSSH (.openssh) or Secure Shell/RFC4716 (.secsh) formats. The public key (.pub) will
be configured for *padmin* via puppet. Add the private key to your client to
connect using pre-shared keys.

### Git

It is assumed the reader has familiarity with git and GitHub, such as how to
fork a repo; how to add ssh-keys for authentication; how to clone, commit, and
push changes. [GitHub.com](http://github.com) has many documents to help learn
git.

## Preparation

Before standing up the lab, you will need to fork or duplicate some repositories.

> The original repositories are public. Any forks will also be public. Some
> data may be considered identifying or at risk and you may wish to use a
> private repo. GitHub has directions to [duplicate a public repo as a
> private repo](https://help.github.com/articles/duplicating-a-repository/)

These repos will become specific to your lab environment and it is **mandatory**
they be forked or duplicated:

  * [controlrepo](https://github.com/puppetinabox/controlrepo)
  * [hiera](https://github.com/puppetinabox/hiera)
  * [lab_config](https://github.com/puppetinabox/lab_config)

The role/profile modules are where a majority of customizations will occur.
Unless you are using puppetinabox 'as-is', you **should** fork/duplicate these repos:

  * [role](https://github.com/puppetinabox/role)
  * [profile](https://github.com/puppetinabox/profile)

These additional repositories will work fine as is until you decide to make
changes. The are designed to be functional as is. Forking/duplicating these
repos is **optional**.

  * [custom_facts](https://github.com/puppetinabox/custom_facts)
  * [linuxfw](https://github.com/puppetinabox/linuxfw)

The repository URLs, whether forks, duplicates, or originals, will be used during
the next step.

### Git Modifications ###

In your *controlrepo production* branch, modify the *git* references in the
**Puppetfile** to point to your fork/duplicate repos:

    # Original
    mod 'role',
      :git => 'git@github.com:puppetinabox/role'
    ...
        
    # New
    mod 'role',
      :git => 'git@github.com:rnelson0/role'
    ...

Modify **r10k_installation.pp** to reference the new controlrepo location:

    # Original
    class { 'r10k':
      version => '1.4.0',
      sources => {
        'puppet' => {
          'remote'  => 'git@github.com:puppetinabox/controlrepo.git',
          'basedir' => "${::settings::confdir}/environments",
          'prefix'  => false,
        },
        'hiera' => {
          'remote'  => 'git@github.com:puppetinabox/hiera.git',
          'basedir' => "/etc/puppet/hiera",
          'prefix'  => false,
        }
      },
      manage_modulepath => false
    }

    # New
    class { 'r10k':
      version => '1.4.0',
      sources => {
        'puppet' => {
          'remote'  => 'git@github.com:rnelson0/controlrepo.git',
          'basedir' => "${::settings::confdir}/environments",
          'prefix'  => false,
        },
        'hiera' => {
          'remote'  => 'git@github.com:rnelson0/hiera.git',
          'basedir' => "/etc/puppet/hiera",
          'prefix'  => false,
        }
      },
      manage_modulepath => false
    }

You may also modify the settings in **hiera.pp** if you desire (optional).

If you are changing the DNS values, modify the *lab_config* repo's
`files/dns/*` files. Hiera data is of course in the *hiera* repo and the
role/profile data are in the *role* and *profile* repos, respectively.

Commit and push the changes upstream:

    git commit -am 'Initial commit of my new puppetized setup'
    # controlrepo
    git push origin production
    # hiera
    git push origin data
    # lab_config, role, profile, custom_facts, linuxfw
    git push origin master

## Bootstrap

The puppet master must be bootstrapped. Create a new node (vm, vagrant box,
bare metal, docker, etc) called *puppet* (suggested IP 10.0.0.5). The entire
bootstrap section will all be performed on this node.  DNS should point the
short name *puppet* to this IP (i.e.  *puppet.example.com* resolves to this
node). If you do not have DNS configured (we will set up DNS later in this
lab), then an **/etc/hosts** entry on nodes will suffice for the moment.

Log in as root. Generate ssh keys for root and add them as [deploy keys]
(https://developer.github.com/guides/managing-deploy-keys/#deploy-keys)
to your repos, or as ssh keys for your Github account. [Set up Git]
(https://help.github.com/articles/set-up-git/#setting-up-git) with the correct
user.name and user.email.

> Reminder: You should mostly be using forks of the puppetinabox repos, but
> the documentation will refer to the original repos. Replace the URI with
> the URI of your fork.

Clone the control repository and cd to its directory:

    git clone git@github.com:puppetinabox/controlrepo.git
    cd controlrepo

### Automated Install

If you know what you are doing or you are not interested in the details of
bootstrapping the master, you may run a script to perform the bootstrap
install.

    ./bootstrap.sh

If the script encounters any issues, proceed step by step with the remaining
instructions and identify the error. Otherwise, you may skip to [Creating
Your First Managed Node] (#creating-your-first-managed-node).

### Manual Install

The manual install is intended for those who want a better understanding of the
bootstrapping process itself, and anyone who encounters issues with the
automated bootstrap script.

Install some modules for the bootstrap process in a temporary location.
[zack/r10k](https://forge.puppetlabs.com/zack/r10k) installs r10k and
[hunner/hiera](https://forge.puppetlabs.com/hunner/hiera) creates the hiera
configuration and directories.

    mkdir -p /root/bootstrap/modules
    puppet module install --modulepath=/root/bootstrap/modules zack/r10k --version 2.5.4
    puppet module install --modulepath=/root/bootstrap/modules stahnma/epel --version 1.0.2
    puppet module install --modulepath=/root/bootstrap/modules stephenrjohnson/puppet --version 1.3.1
    puppet module install --modulepath=/root/bootstrap/modules hunner/hiera --version 1.1.1

Apply the puppet configuration

    puppet apply --modulepath=/root/bootstrap/modules master.pp

Apply the hiera configuration:

    puppet apply --modulepath=/root/bootstrap/modules hiera.pp

Apply the configuration with with:

    puppet apply --modulepath=/root/bootstrap/modules r10k_installation.pp

This will install r10k and configure it to use your defined *controlrepo*. You
can then run r10k as root:

    r10k deploy environment -p

This will create a puppet environment called production at
**/etc/puppet/environments/production** with all of the modules specified in
the *controlrepo* **Puppetfile**, including the other repos that you forked.
The hiera repository will be checked out to **/etc/puppet/hiera/data**. Puppet
is ready for its first run.  You may preview what will be applied to the puppet
master with the noop flag:

    puppet agent -t --noop

You can then apply the catalog by dropping the noop flag:

    puppet agent -t

> You may have to run the above command twice. The initial setup involves
> creating a database, populating it, and starting the database service, which
> can sometimes take longer than the puppet's timeout. Don't worry, just
> run the command again and it will complete on the second try.

Lastly, ensure that the *puppet* service (the agent) is running and set to run
at startup. On an EL linux, use the following commands:

    chkconfig puppet on
    service puppet start

Congratulations! You now have a fully functioning puppet server that supports
puppet with Apache/Passenger for scalability, hiera for external data, and
puppetdb for exported resources and reporting.

## Creating Your First Managed Node

You can now create additional nodes and assign services via puppet. Included is
a [custom_facts module](https://github.com/puppetinabox/custom_facts) and site
manifest that assigns roles to nodes based on the hostname. If the hostname
matches the format *role[##]*, the role is applied. For instance, our node
called *puppet* received the role called *puppet*. Another node named *puppet2*
would receive the same role. The provided roles are:

  * dns: single server, can handle multiple zones via hiera
  * dhcp: single server, can handle multiple scopes and reservations via hiera
  * puppet: all in one puppet master, hiera, and puppetdb roles
  * build: includes FPM, rspec-puppet and puppetlabs_spec_helper (via RVM for CentOS 6.5), gcc, and rpmbuild. No managed git server is provided, the use of GitHub is encouraged for persistent repositories.
  * yumrepo: Create yum repositories based on hiera data. Manages underlying directory structure as well
  * webserver: generic Apache web server role. It can be improved upon or used as a base to build more specific roles targeted to specific apps
  * mysql: MySQL database server with client. **Requires an additional 41GB of unused space on the existing drive ([more info](http://rnelson0.com/2020/01/31/deploying-mysql-with-puppet-without-disabling-selinux/)).**
  * phpmyadmin: phpMyAdmin, a web interface for a MySQL database
  * tftp: TFTP server for PXE boot/Auto Deploy or to host network device firmware updates.

DNS is required for almost everything, which makes it a great starting point.
Without it, every node will need hosts entries, as we had to add to the
*puppet* node during bootstrapping. This is the second node you should create.

Create a new node called *dns* or *dns01* (suggested IP 10.0.0.253). Because
DNS is not available yet, be sure to apply the hosts entries from the
controlrepo before continuing. Use SCP, git clone, or just cat/vi to create
the file on the new node. Modify the IPs in the file first if you are not
using 10.0.0.0/24:

    puppet apply hosts_add.pp

Next, we will assign it a zone file. This is done via hiera and the *lab_config*
module. The provided hiera in the *controlrepo* will generate a zone file for
*example.com*, using the contents of [files/dns/example.com]
(https://github.com/puppetinabox/lab_config/blob/master/files/dns/example.com)
(forward) and [files/dns/0.0.10]
(https://github.com/puppetinabox/lab_config/blob/master/files/dns/0.0.10)
(backward) from *lab_config*. If you are using the example network, then no
changes are required.  If you are using another network, adjust the contents of
these files and commit/push the changes to the master branch of *lab_config*
and re-run r10k, adding the environment name this time:

    r10k deploy environment production -p

On the new *dns* node, run puppet:

    puppet agent -t

On the *puppet* node, a certificate will be generated that needs signed. Use the
following commands to list the cert and then sign it with the full name (ex:
*dns.example.com*):

    puppet cert list
    puppet cert sign "dns.example.com"

Run the agent on *dns* again:

    puppet agent -t

> There is an outstanding bug with the thias/bind module related to resource
> ordering. Until the PR is merged, you will need to run `puppet agent -t`
> once more to complete the setup.

Now that the cert is signed, re-run puppet on *dns* and the DNS service
will be installed, configured, provided the zone file(s), and started. You can
now point a DNS client at the the IP of *dns* and resolve *puppet*, *dns*, and
other records as provided in the zone files.

```
[root@dns ~]# nslookup
> server 127.0.0.1
Default server: 127.0.0.1
Address: 127.0.0.1#53
> build.example.com
Server:         127.0.0.1
Address:        127.0.0.1#53

Name:   build.example.com
Address: 10.0.0.10
> exit
```

You should update your VM
templates to point to the IP of *dns*. You should also switch to *padmin* and
use sudo instead of direct root access.

Finally, enable the puppet service. This will force the node to check in once
per interval (default 30m):

    sudo chkconfig puppet on
    sudo service puppet start 

To update the zone files, changes are made to the contents in the *lab_config*
repository (and pushed upstream!), r10k is rerun on the master, and puppet is
rerun on *dns*. If the changes are not urgent, the last step can be skipped
and *dns* will pick up the changes on its next checkin, within 30 minutes.

Now that the DNS service exists, you can remove the hosts entries on *dns*
using the **hosts_remove.pp** file from the controlrepo:

    sudo puppet apply hosts_remove.pp

## Additional Nodes

The *controlrepo* includes sample hiera data for additional nodes: *dhcp*
(10.0.0.254), *build* (10.0.0.???), and *yumrepo* (10.0.0.252). Any files
required are in *lab_config*. There is also a role for *webserver*, which does
not require any hiera configuration. As with DNS, you can deploy these services
by creating a node with the same name as the role, with an optional trailing
number, i.e. *webserver02*, *build1*, or *dhcp*.

After you create the yum repo, change the 'lab' repo in [manifests/base.pp]
(https://github.com/puppetinabox/profile/blob/master/manifests/base.pp#L56) to
`enable => 1,`

## Get Labbing

With your base lab nodes in place, you can now start using the lab. Create your
own roles or use the lab as a testbed for proof of concepts. I would to hear
how you use your lab, find me on 
[twitter](https://twitter.com/rnelson0) or open an
[issue](https://github.com/puppetinabox/documentation/issues)!

### Roadmap

View the [planned milestones](https://github.com/puppetinabox/documentation/milestones).

### BUGS

The following issues are known bugs:

  * The *puppet* and *dns* nodes require two consecutive runs in most cases.
