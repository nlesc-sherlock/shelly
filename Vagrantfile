# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.network "public_network", bridge: "eth0"

  config.vm.define :master do |master|
    master.vm.hostname = "shelly.#{ENV['SHELLY_DOMAIN']}"
  end

  (1..2).each do |i|
    config.vm.define "slave#{i}" do |node|
      node.vm.hostname = "shelly#{i}.#{ENV['SHELLY_DOMAIN']}"
    end
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4196"
  end

  config.vm.provision "fix-no-tty", type: "shell" do |s|
    s.privileged = false
    s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
  end

  config.vm.provision "shell", inline: <<-SHELL
    mkdir -p /root/.ssh
    cp /vagrant/shelly.key /root/.ssh/id_rsa
    cp /vagrant/shelly.key.pub /root/.ssh/id_rsa.pub
    cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
    chmod -R 600 /root/.ssh
  SHELL
end
