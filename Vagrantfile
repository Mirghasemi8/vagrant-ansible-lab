
# Set VMware Desktop as the default Vagrant provider.
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'vmware_desktop'

# Define the base Vagrant box (Debian 12).
ENV['DEFAULT_DISTRO'] = 'bento/debian-12'

# Define the number of managed nodes to create.
ENV['NODE_COUNT'] = "3"


Vagrant.configure("2") do |config|

  # Set the base operating system for all virtual machines.
  config.vm.box = ENV['DEFAULT_DISTRO']

  # Configure VMware resources shared by all virtual machines.
  config.vm.provider "vmware_desktop" do |v|
    v.memory = 1000  # Allocate 1000 MB of RAM.
    v.cpus = 1       # Allocate one virtual CPU.
  end


  # --------------------------------------------------
  # Control Node Configuration
  # --------------------------------------------------

  # Define the Ansible control node.
  config.vm.define "control-node" do |control|

    # Set the hostname inside the virtual machine.
    control.vm.hostname = "control-node"

    # Assign a static private IP address.
    control.vm.network "private_network", ip: "192.168.56.10"

    # Install Ansible on the control node.
    # This provisioner runs with root privileges.
    control.vm.provision "shell", inline: <<~SHELL

      # Prevent interactive prompts during package installation.
      export DEBIAN_FRONTEND=noninteractive

      # Refresh the APT package index.
      apt-get update

      # Install Ansible.
      apt-get install -y ansible

      # Add the ANSIBLE_CONFIG environment variable if it
      # does not already exist in /etc/environment.
      if ! grep -qF "ANSIBLE_CONFIG=/home/vagrant/ansible/ansible.cfg" /etc/environment; then
        echo "ANSIBLE_CONFIG=/home/vagrant/ansible/ansible.cfg" >> /etc/environment
      fi

    SHELL

    # Share the local Ansible directory with the control node.
    control.vm.synced_folder "./ansible", "/home/vagrant/ansible"

    # Share the local SSH keys directory with the control node.
    control.vm.synced_folder "./.keys", "/home/vagrant/.keys"


    # Generate an SSH key pair for Ansible authentication.
    # Run as the vagrant user rather than root.
    control.vm.provision "shell", inline: <<~SHELL, privileged: false

      # Generate the key only if it does not already exist.
      if [ ! -f '/home/vagrant/.ssh/ansible-key' ]; then

        # Generate an Ed25519 key pair without a passphrase.
        ssh-keygen -t ed25519 -f /home/vagrant/.ssh/ansible-key -qN ""

        # Copy the public key to the shared keys directory.
        cp /home/vagrant/.ssh/ansible-key.pub /home/vagrant/.keys
      fi

    SHELL


    # Define a manually triggered provisioner to install Docker.
    # It will not run automatically during vagrant up.
    control.vm.provision "install-docker",
      type: "shell",
      run: "never",
      privileged: false,
      inline: <<~SHELL

      # Specify the Ansible configuration file.
      export ANSIBLE_CONFIG=/home/vagrant/ansible/ansible.cfg

      # Navigate to the Ansible project directory and execute the playbook.
      cd /home/vagrant/ansible/ && ansible-playbook install-docker.yml

    SHELL

  end


  # --------------------------------------------------
  # Managed Nodes Configuration
  # --------------------------------------------------

  # Dynamically create the number of nodes specified by NODE_COUNT.
  (1..ENV['NODE_COUNT'].to_i).each do |i|

    # Define each managed node using its sequential number.
    config.vm.define "node-#{i}" do |node|

      # Set the hostname (node-1, node-2, node-3, etc.).
      node.vm.hostname = "node-#{i}"

      # Assign a unique static IP address to each node.
      # Example: node-1 = 192.168.56.11
      #          node-2 = 192.168.56.12
      #          node-3 = 192.168.56.13
      node.vm.network "private_network", ip: "192.168.56.#{10 + i}"

      # Share the public keys directory with each managed node.
      node.vm.synced_folder "./.keys", "/home/vagrant/.keys"


      # Configure SSH authentication for the Ansible control node.
      # Run as the vagrant user rather than root.
      node.vm.provision "shell", inline: <<~SHELL, privileged: false

        # Check whether the control node's public key is available.
        if [ -f '/home/vagrant/.keys/ansible-key.pub' ]; then

          # Add the public key to authorized_keys only if
          # it has not already been added.
          if ! grep -qF "$(cat /home/vagrant/.keys/ansible-key.pub)" /home/vagrant/.ssh/authorized_keys; then

            # Authorize the control node to connect via SSH.
            cat /home/vagrant/.keys/ansible-key.pub >> /home/vagrant/.ssh/authorized_keys
          fi
        fi

      SHELL

    end
  end

end
