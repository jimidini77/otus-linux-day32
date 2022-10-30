# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :router1 => {
    :box_name => "centos/7",
    :vm_name => "router1",
    :net => [
      {ip: '10.0.10.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"}, # /29
      {ip: '10.0.12.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"}, # /29
      {ip: '192.168.10.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net1"}, # /29
      {ip: '192.168.56.101', adapter: 5}
    ]
  },

  :router2 => {
    :box_name => "centos/7",
    :vm_name => "router2",
    :net => [
      {ip: '10.0.10.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"}, # /29
      {ip: '10.0.11.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"}, # /29
      {ip: '192.168.20.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net2"}, # /29
      {ip: '192.168.56.102', adapter: 5}
    ]
  },

  :router3 => {
    :box_name => "centos/7",
    :vm_name => "router3",
    :net => [
      {ip: '10.0.11.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"}, # /29
      {ip: '10.0.12.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"}, # /29
      {ip: '192.168.30.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net3"}, # /29
      {ip: '192.168.56.103', adapter: 5}
    ]
  },
  
  :master => {
    :box_name => "centos/7",
    :vm_name => "master",
    :net => [
      {ip: '192.168.56.100', adapter: 2}
    ]
  },
}

Vagrant.configure("2") do |config|
  config.vm.synced_folder ".", "/vagrant", disabled: true
  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s

      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", ipconf
      end
      
      if boxconfig.key?(:public)
        box.vm.network "public_network", boxconfig[:public]
      end

      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
              cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL

      case boxname.to_s
      when "master"
        box.vm.provision "shell", run: "always", inline: <<-SHELL
          sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config && systemctl restart sshd.service
          yum install -y epel-release
          yum install -y ansible
          mkdir /study/
          chmod g+w /study/
          gpasswd -a vagrant root          
          SHELL
      # when "inetRouter2"
      #   box.vm.network "forwarded_port", guest:8080, host: 8080
      when "router3"
        box.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/provision.yml"
          ansible.inventory_path = "ansible/hosts"
          ansible.host_key_checking = "false"
          ansible.limit = "all"
        end
      end
    end
  end
end
