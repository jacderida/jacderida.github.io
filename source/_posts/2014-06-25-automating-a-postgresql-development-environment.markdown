---
layout: post
title: "Automating a PostgreSQL development environment with Packer and Vagrant"
date: 2014-06-25 23:31:03 +0100
comments: true
categories:
- automation
- packer
- vagrant
- postgres
---

In this post, I'm going to describe how to use Packer and Vagrant to automate a machine running PostgreSQL. Rather than repeating it constantly, I'll state here at the start, that the resulting machine is *not* intended to be used in a production environment. It should be patently obvious that with respect to security, it's pretty lax. This post also isn't intended to be a general introduction to Packer and Vagrant; it assumes you're already vaguely familiar with those tools.

I wanted a machine to use in a local dev environment, so I'll use Packer to produce a VirtualBox image for use with Vagrant. I chose CentOS for the host OS, although that choice was somewhat arbitrary; PostgreSQL supports most Linux based distributions, and it can also be run on Windows, or even Cygwin. Doing a VirtualBox based image with Packer involves doing an unattended CentOS installation, which is based on the Red Hat [Kickstart](https://www.centos.org/docs/5/html/Installation_Guide-en-US/pt-install-advanced-deployment.html) installation method.

For full reference, the entire setup is available in my [automation repository](https://github.com/jacderida/automation/tree/master/box_templates/postgres-CentOS-6.5-x86_64). To give everything a bit more context, it'll probably be handy to refer to that. In this post, I'll just go through the noteworthy parts. First, a look at the VirtualBox image definition in the packer template.
``` json template.json
"builders": [
    {
        "type": "virtualbox-iso",
        "boot_command": [ "<tab> text ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ks.cfg<enter><wait>" ],
        "boot_wait": "10s",
        "disk_size": 10140,
        "guest_os_type": "RedHat_64",
        "http_directory": "http",
        "iso_checksum": "0d9dc37b5dd4befa1c440d2174e88a87",
        "iso_checksum_type": "md5",
        "iso_url": "CentOS-6.5-x86_64-minimal.iso",
        "ssh_username": "vagrant",
        "ssh_password": "vagrant",
        "ssh_port": 22,
        "ssh_wait_timeout": "10000s",
        "shutdown_command": "echo '/sbin/halt -h -p' > shutdown.sh; echo 'vagrant' | sudo -S sh 'shutdown.sh'",
        "guest_additions_path": "/tmp/VBoxGuestAdditions_{{.Version}}.iso",
        "virtualbox_version_file": "/tmp/.vbox_version",
        "vboxmanage": [
            [ "modifyvm", "{{.Name}}", "--memory", "512" ],
            [ "modifyvm", "{{.Name}}", "--cpus", "1" ]
        ]
    }
]

```
Nothing much to point out, other than we're using the minimal install for CentOS, which comes in at a compact 400MB. Noteworthy is that the configuration is assuming the ISO is in the same directory as template.json. You can put a full web based URL in there, or a network location (which often comes in useful in a corporate environment). Unfortunately, my internet connection tends to be pretty unreliable these days, so I'm making the assumption that the ISO has been downloaded in advance (of course, an ISO wouldn't be committed to the repository).

Here's a look at the Kickstart file, ks.cfg, located in the http directory.

``` bash ks.cfg
install
cdrom
lang en_GB
keyboard uk
network --bootproto=dhcp --hostname=postgres-CentOS-6-5-x86_64
rootpw vagrant
firewall --enabled --service=ssh
authconfig --enableshadow --passalgo=sha512
selinux --disabled
timezone --utc Europe/Dublin
bootloader --location=mbr
text
skipx
zerombr
clearpart --all --initlabel
autopart
auth  --useshadow  --enablemd5
firstboot --disabled
reboot

%packages --nobase
@core
%end

%post
sed -i /etc/sysconfig/network-scripts/ifcfg-eth0 -e "s/^ONBOOT=no/ONBOOT=yes/g"
sed -i /etc/sysconfig/network-scripts/ifcfg-eth0 -e "s/^NM_CONTROLLED=yes/NM_CONTROLLED=no/g"
service network restart
/usr/bin/yum -y update
/usr/bin/yum -y install sudo
/usr/bin/yum -y install wget
/usr/sbin/groupadd vagrant
/usr/sbin/useradd vagrant -g vagrant -G wheel
echo "vagrant" | passwd --stdin vagrant
echo "vagrant        ALL=(ALL)       NOPASSWD: ALL" >> /etc/sudoers.d/vagrant
chmod 0440 /etc/sudoers.d/vagrant

/usr/sbin/useradd postgres -G wheel
echo "postgres" | passwd --stdin postgres

sed -i "s/^.*requiretty/#Defaults requiretty/" /etc/sudoers
echo "UseDNS no" >> /etc/ssh/sshd_config
%end
```

We can see there are various things customised for the UK environment, where I happen to be located. To adjust those based on your own location, you'd need to refer to the documentation to discern the valid values. In the %packages section, you get the opportunity to specify any individual packages, or groups of packages, available on the repository that comes with the particular installation. If we were doing the install with the DVD rather than the CD, the packages and groups available would be vastly larger. In this case, we'll chose the absolute bare minimum installation. Just to add as a bit of a side note, if you're using the larger DVD install, it's not immediately obvious how to establish what the valid values are for packages and groups. To do that, you pretty much need a pre-existing machine with yum on it. In terms of the groups of packages, from memory, I can't remember how to use yum to search for them.
