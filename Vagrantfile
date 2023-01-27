# Describe VMs
MACHINES = {
  # VM name "kernel update"
  :"nfssrv" => {
              # VM box
              :box_name => "generic/debian10",
              # VM CPU count
              :cpus => 1,
              # VM RAM size (Mb)
              :memory => 1024,
              # networks
              :net => ["192.168.56.10"],
              # forwarded ports
              :forwarded_port => [],
              # configure script
              :configurer => "deployNFSSrv.yml"
  },
  :"nfscl" => {
              # VM box
              :box_name => "generic/debian10",
              # VM CPU count
              :cpus => 1,
              # VM RAM size (Mb)
              :memory => 1024,
              # networks
              :net => ["192.168.56.11"],
              # forwarded ports
              :forwarded_port => [],
              # configure script
              :configurer => "deployNFSCl.yml"
  }
}

Vagrant.configure("2") do |config|
  # Disable shared folders
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.box_check_update = false

  MACHINES.each do |boxname, boxconfig|
    # Apply VM config
    config.vm.define boxname do |box|
      # Set VM base box and hostname
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      # Additional network config if present
      if boxconfig.key?(:net)
        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ip: ipconf
        end
      end
      # Port-forward config if present
      if boxconfig.key?(:forwarded_port)
        boxconfig[:forwarded_port].each do |port|
          box.vm.network "forwarded_port", port
        end
      end
      # VM resources config
      box.vm.provider "virtualbox" do |v|
        # Set VM RAM size and CPU count
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
#        boxconfig[:disks].each do |dname, dconf|
#          unless File.exist?(dconf[:dfile])
#	    v.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
#            v.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
#          end
#        end
      end
      if boxconfig[:configurer].length > 0
      box.vm.provision "ansible" do |ansible|
        ansible.playbook = boxconfig[:configurer]
        ansible.verbose = "v"
      end
#        box.vm.provision "shell", path: boxconfig[:configurer]
#
      end
    end
  end
end

