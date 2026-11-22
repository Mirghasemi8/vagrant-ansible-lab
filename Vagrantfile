ENV['VAGRANT_DEFAULT_PROVIDER'] = 'vmware_desktop'
ENV['DEFAULT_DISTRO'] = 'bento/debian-12'
ENV['DEFAULT_NODE_COUNTS'] = '3'

Vagrant.configure("2") do |config|

  config.vm.box = ENV['DEFAULT_DISTRO']
  config.vm.provider "vmware_desktop" do |v|
    v.memory = 1000
    v.cpus = 1
  end

  config.vm.define "control-node" do |control|
    control.vm.hostname = "control-node"
    control.vm.network "private_network", ip: "192.168.56.10"

    control.vm.provision "shell", inline: <<~SHELL
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y ansible
    SHELL

    control.vm.synced_folder "./ansible", "/home/vagrant/ansible"
    control.vm.synced_folder "./.keys", "/home/vagrant/.keys"

    control.vm.provision "shell", inline: <<~SHELL, privileged: false
    if [ ! -f '/home/vagrant/.ssh/ansible-key' ]; then 
      ssh-keygen -t ed25519 -f /home/vagrant/.ssh/ansible-key -qN ""
      cp /home/vagrant/.ssh/ansible-key.pub /home/vagrant/.keys
    fi
    SHELL
  end


  config.vm.define "node" do |node|
    node.vm.hostname = "node"
    node.vm.network "private_network", ip: "192.168.56.11"

    node.vm.synced_folder "./.keys", "/home/vagrant/.keys"

    node.vm.provision "shell", inline: <<~SHELL, privileged: false
    if [ -f '/home/vagrant/.keys/ansible-key.pub' ]; then
    if ! grep -qF "$(cat /home/vagrant/.keys/ansible-key.pub)" /home/vagrant/.ssh/authorized_keys ; then
    cat /home/vagrant/.keys/ansible-key.pub >> /home/vagrant/.ssh/authorized_keys
    fi
    fi
    SHELL
  end



end
