Vagrant.configure("2") do |config|
    IMAGE = "ubuntu/jammy64"

    NODES = [
      { name: "node1", ip: "192.168.56.11", ram: 2048, disk: "15GB" },
      { name: "node2", ip: "192.168.56.12", ram: 2048, disk: "15GB" },
      { name: "node3", ip: "192.168.56.13", ram: 2048, disk: "15GB" }
    ]

  NODES.each do |node|
    config.vm.define node[:name] do |node_config|
      node_config.vm.box = IMAGE
      node_config.vm.hostname = node[:name]
      node_config.vm.network "private_network", ip: node[:ip]

      # Set disk size (plugin required)
      node_config.disksize.size = node[:disk]

      node_config.vm.provider "virtualbox" do |vb|
        vb.memory = node[:ram]
        vb.cpus = 1
      end

      # Provisioning: Set username, password, grant sudo privileges, and add SSH key
      node_config.vm.provision "shell", inline: <<-SHELL
        #!/bin/bash
        # Set the password for the ubuntu user
        echo "ubuntu:ubuntu" | chpasswd
        # Ensure the ubuntu user has sudo privileges
        usermod -aG sudo ubuntu
        # Create the .ssh directory for the ubuntu user
        mkdir -p /home/ubuntu/.ssh
        # Add the public SSH key to authorized_keys
        echo "#{File.read('~/.ssh/id_rsa.pub')}" >> /home/ubuntu/.ssh/authorized_keys
        # Set the correct permissions
        chown -R ubuntu:ubuntu /home/ubuntu/.ssh
        chmod 700 /home/ubuntu/.ssh
        chmod 600 /home/ubuntu/.ssh/authorized_keys
      SHELL
    end
  end
end
