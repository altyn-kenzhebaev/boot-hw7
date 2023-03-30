# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME'] 

MACHINES = {
    :repo => {
        :box_name => "centos/7",
        :box_version => "1804.02",
        #:ip_addr => '192.168.50.10',
        #:script => 'rpm_repo_create.sh',
        :cpus => 1,
        :memory => 512,
        :disks => {
            :sata1 => {
                :dfile => home + '/VirtualBox VMs/rep/sata1.vdi',
                :size => 40860,
                :port => 1
            },
        }
    },
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
      config.vm.define boxname do |box|
          box.vm.box = boxconfig[:box_name]
          box.vm.box_version = boxconfig[:box_version]
          box.vm.host_name = boxname.to_s #+ ".test.local"
          #box.vm.network "private_network", ip: boxconfig[:ip_addr], virtualbox__intnet: "net1"
          box.vm.provider :virtualbox do |vb|
            vb.memory = boxconfig[:memory]
            vb.cpus = boxconfig[:cpus]
            needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                    vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                    needsController =  true
                end
            end
            if needsController == true
              vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                boxconfig[:disks].each do |dname, dconf|
                  vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                end
            end
          end
          #box.vm.provision "shell", path: boxconfig[:script]
      end
  end
end
