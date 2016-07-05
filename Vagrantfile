# -*- mode: ruby -*-
# vi: set ft=ruby :

$script = <<SCRIPT
  apt-get update -q
  apt-get install -q -y ruby2.0 ruby2.0-dev zlib1g-dev nodejs
  gem2.0 install bundler
  sudo -u vagrant bash -c '\
    export LC_ALL=C; \
    cd /vagrant; \
    bundler install'
  # Notes:
  #   - cd /vagrant; jekyll serve --host=0.0.0.0
  #   - http://192.168.100.101:4000
SCRIPT

Vagrant.configure('2') do |config|
  config.vm.box = 'ubuntu/trusty64'
  config.vm.hostname = 'dev'
  config.vm.provider :virtualbox do |vb|
    vb.customize [
      'modifyvm', :id,
      '--memory', '1024',
      '--natdnshostresolver1', 'on',
      '--accelerate3d', 'off',
      '--name', 'www.carlosabalde.com',
    ]
  end
  config.vm.provision :shell, :privileged => true, :keep_color => false, :inline => $script
  config.vm.network :public_network
  config.vm.network :private_network, ip: '192.168.100.101'
  config.vm.synced_folder '.', '/vagrant', :nfs => false
end
