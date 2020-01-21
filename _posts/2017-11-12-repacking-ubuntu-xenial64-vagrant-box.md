---
layout: post
title: "Repacking an ubuntu/xenial64 Vagrant box"
date: 2017-11-12 15:30:00 +0100
tags:
  - english
  - dev
---

I use Vagrant + VirtualBox + SaltStack a lot in order to create lightweight & reproducible development environments. I tend to use custom boxes based on popular public boxes like `ubuntu/trusty64`. Usually my boxes are simple mirrors, but due to some unpleasing experiences in the past I prefer to always keep my own copies and completely freeze my development environments.

Creating a mirror of an existing box is trivial: minimal Vagrant file disabling the `ssh.insert_key` feature + `vagrant up` + `vagrant package` + upload to [https://app.vagrantup.com](https://app.vagrantup.com). I've done that many times and it works like a charm. To be more precise, it has been working like a charm until today, when I tried to create a mirror of the `ubuntu/xenial64` box.

<!--more-->

Of course I'm not the first one experiencing this problem. The symptoms have been extensible described in [https://github.com/hashicorp/vagrant/issues/5186/](https://github.com/hashicorp/vagrant/issues/5186/). However, I was not able to find any article clearly describing the solution to this headache. Therefore, here it goes the recipe you're looking for :)

1. Do the usual thing: create a minimal Vagrant file, but don't worry about the `ssh.insert_key` feature this time. Let Vagrant secure the VM during the initial boot. We'll revert that later.

        Vagrant.configure('2') do |config|
          config.vm.box = 'ubuntu/xenial64'
          config.vm.box_version = '=20171110.0.0'
          config.vm.provider 'virtualbox' do |vb|
            vb.customize [ 'modifyvm', :id, '--uartmode1', 'disconnected' ]
          end
          # config.ssh.insert_key = false
          # config.ssh.username = 'vagrant'
          config.vbguest.auto_update = false
        end

2. Create the `vagrant` user, add it to the same groups assigned to the `ubuntu` user, set up its SSH key using the insecure Vagrant key pair, etc. Basically, you need to follow the steps in [https://www.vagrantup.com/docs/boxes/base.html](https://www.vagrantup.com/docs/boxes/base.html) to fix what's missing in the base box.

        $ sudo adduser vagrant
        $ sudo usermod -a -G ubuntu vagrant
        $ sudo usermod -a -G adm vagrant
        $ ...
        $ sudo usermod -a -G lxd vagrant
        $ sudo -u vagrant mkdir /home/vagrant/.ssh/
        $ sudo -u vagrant chmod 700 /home/vagrant/.ssh/
        $ sudo -u vagrant wget --no-check-certificate \
            https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub \
            -O /home/vagrant/.ssh/authorized_keys
        $ sudo -u vagrant chmod 600 /home/vagrant/.ssh/authorized_keys
        $ sudo visudo
            # vagrant ALL=(ALL) NOPASSWD: ALL
        $ sudo passwd root

3. Ungracefully stop the VM (for example using the VirtualBox UI) and then adjust the Vagrant file uncommenting the SSH options initially disabled. Next, remove the random private key stored in the host machine during the initial boot (i.e. `rm .vagrant/machines/default/virtualbox/private_key`).

4. Execute a `vagrant up + ssh + halt` cycle in order to check everything is working as expected. Then you can package the box in the usual way (i.e. `vagrant package`) and destroy the VM (i.e. `vagrant destroy`).

5. You may want to check everything works as expected in preparation for the upload of the box to [https://app.vagrantup.com](https://app.vagrantup.com). You can do it the usual way:

        $ vagrant box add --name foo /path/to/package.box
        $ vagrant up
            Vagrant.configure('2') do |config|
              config.vm.box = 'foo'
            end
        $ VAGRANT_LOG=info vagrant ssh
        $ ...
        $ vagrant destroy
        $ vagrant box remove foo

6. That's all you need to know. It's not rocket science at all, but this is the kind of thing you expect to work out of the box (pun intended :) and it becomes a headache when things don't go as expected.

Hope it helps! :)
