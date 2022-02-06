require 'fileutils'

# file operations needs to be relative to this file
VAGRANT_ROOT = File.dirname(File.expand_path(__FILE__))

# directory that will contain VDI files
VAGRANT_DISKS_DIRECTORY = "disks"

# controller definition
VAGRANT_CONTROLLER_NAME = "Virtual I/O Device SCSI controller"
VAGRANT_CONTROLLER_TYPE = "virtio-scsi"

# define disks
# The format is filename, size (GB), port (see controller docs)
local_disks = [
  { :filename => "disk1", :size => 15, :port => 5 },
  { :filename => "disk2", :size => 20, :port => 6 },
]

ENV["LC_ALL"] = "en_US.UTF-8"

Vagrant.configure("2") do |config|

    disks_directory = File.join(VAGRANT_ROOT, VAGRANT_DISKS_DIRECTORY)

    # create disks before "up" action
    config.trigger.before :up do |trigger|
        trigger.name = "Create disks"
        trigger.ruby do
            unless File.directory?(disks_directory)
                FileUtils.mkdir_p(disks_directory)
            end
            local_disks.each do |local_disk|
                local_disk_filename = File.join(disks_directory, "#{local_disk[:filename]}.vdi")
                unless File.exist?(local_disk_filename)
                    puts "Creating \"#{local_disk[:filename]}\" disk"
                    system("vboxmanage createmedium --filename #{local_disk_filename} --size #{local_disk[:size] * 1024} --format VDI")
                end
            end
        end
    end

    config.vm.define "k8s-master" do |node|
        node.vm.box = "ubuntu/focal64"
        node.vm.network "private_network", ip: "192.168.77.10"
        node.vm.provider "virtualbox" do |v|
            v.name = "k8s-master-01"
            v.memory = 4096
            v.cpus = 2

            # create storage controller on first run
            unless File.directory?(disks_directory)
                v.customize ["storagectl", :id, "--name", VAGRANT_CONTROLLER_NAME, "--add", VAGRANT_CONTROLLER_TYPE, '--hostiocache', 'off']
            end

            # attach storage devices
            local_disks.each do |local_disk|
                local_disk_filename = File.join(disks_directory, "#{local_disk[:filename]}.vdi")
                unless File.exist?(local_disk_filename)
                    v.customize ['storageattach', :id, '--storagectl', VAGRANT_CONTROLLER_NAME, '--port', local_disk[:port], '--device', 0, '--type', 'hdd', '--medium', local_disk_filename]
                end
            end
        end
        # node.disksize.size = "30GB"
        node.vm.provision "ansible" do |ansible|
            ansible.galaxy_role_file = "provision/requirements.yml"
            ansible.playbook = "provision/playbook.yml"
            ansible.extra_vars = {
                ntp_timezone: "Asia/Ho_Chi_Minh"
            }
        end
    end

    # cleanup after "destroy" action
    config.trigger.after :destroy do |trigger|
        trigger.name = "Cleanup operation"
        trigger.ruby do
            # the following loop is now obsolete as these files will be removed automatically as machine dependency
            local_disks.each do |local_disk|
                local_disk_filename = File.join(disks_directory, "#{local_disk[:filename]}.vdi")
                if File.exist?(local_disk_filename)
                    puts "Deleting \"#{local_disk[:filename]}\" disk"
                    system("vboxmanage closemedium disk #{local_disk_filename} --delete")
                end
            end
            if File.exist?(disks_directory)
                FileUtils.rmdir(disks_directory)
            end
        end
    end
end