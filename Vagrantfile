# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
    config.vm.define "backup_srv" do |srv|
      srv.vm.box = "centos/7"
      srv.vm.box_check_update = false
      srv.vm.hostname = "backup"
      srv.vm.network "private_network", ip: "192.168.11.160"
      srv.vm.provider "virtualbox" do |vb|
        second_disk = "disk.vdi"
        unless File.exist?("disk.vdi")
          vb.customize ['createhd', '--filename', second_disk, '--size', 3 * 1024]
        end
        vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', second_disk]
      end
    end
  
    config.vm.define "client" do |client|
        client.vm.box = "centos/7"
        client.vm.box_check_update = false
        client.vm.hostname = "client"
        client.vm.network "private_network", ip: "192.168.11.150"
    end
  
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "back.yml"
    end
  end