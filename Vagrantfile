Vagrant.configure("2") do |config|
  config.vm.define "soufe1" do |soufe1|
    soufe1.vm.hostname = "soufe1"
    soufe1.vm.box = "almalinux/9"
    soufe1.vm.network "private_network", ip: "192.168.10.101", netmask: "255.255.0.0"
    soufe1.vbguest.auto_update = false
    soufe1.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbook.yml"
      ansible.tags = "soufe1"
    end
  end

  config.vm.define "soufe2" do |soufe2|
    soufe2.vm.hostname = "soufe2"
    soufe2.vm.box = "almalinux/9"
    soufe2.vm.network "private_network", ip: "192.168.10.102", netmask: "255.255.0.0"
    soufe2.vbguest.auto_update = false
    soufe2.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbook.yml"
      ansible.tags = "soufe2"
    end
  end
end
