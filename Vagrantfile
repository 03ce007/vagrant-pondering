# minimum vagrant api
VAGRANT_API_VERSION = "2"

DEFAULT_BOX="bento/centos-7.7"
DEFAULT_BOX_VERSION="202004.15.0"

# resource allocation for candidate hosts
HOST_MEM_HIGH_VAL=8192
HOST_MEM_MID_VAL=2048
HOST_MEM_LOW_VAL=1024
HOST_CPU_HIGH_VAL=8
HOST_CPU_MID_VAL=4
HOST_CPU_LOW_VAL=2

HOST_DOMAIN = "vagrant.ponder"

# vms
ANSIBLE_MASTER_VM="ansible0"    # DO NOT DELETE or CHANGE
# add more prefixes as needed
SERVER_VM="server0"
CLIENT_VM="client0"


PONDERING_POND = [
    # NOTE:
    #   1 - ensure to check below flags in each entry
    #       - ansible_enabled (bool) is set to true for ANSIBLE_MASTER_VM only
    #       - forwarded_ports (key/value) = exposing ports, if needed
    # an example
    { :hostname => "#{SERVER_VM}1", :ip => "10.11.22.100", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false },
    { :hostname => "#{CLIENT_VM}1", :ip => "10.11.22.101", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false },
    { :hostname => "#{CLIENT_VM}2", :ip => "10.11.22.102", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false, :forwarded_ports => [{ :guest => 8500, :host => 8500 }] },
    # ensure that ansible master is the last one
    { :hostname => "#{ANSIBLE_MASTER_VM}1", :ip => "10.99.1.100", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => true, :forwarded_ports => [] },
]
#####################################################################
### add any common configuration needed on all hosts
### eg., sync /etc/hosts, disable SELinux, etc
#####################################################################
ETC_HOSTS_ENTRIES = ""
KNOWN_HOSTS_ENTRIES = ""
PONDERING_POND.each do |n|
    ETC_HOSTS_ENTRIES << "#{n[:ip]} #{n[:hostname]}.#{HOST_DOMAIN} #{n[:hostname]}\n"
    # only applicable/usable for ansible master vm
    KNOWN_HOSTS_ENTRIES << "ssh-keyscan #{n[:hostname]}.#{HOST_DOMAIN},#{n[:ip]} >> /home/vagrant/.ssh/known_hosts"
end

# hold script to update /etc/hosts into a variable
$DEFAULT_SHELL_SCRIPT = <<SCRIPT
#!/bin/bash

echo "SHELL PROVISIONING: Updating /etc/hosts file"
cat > /etc/hosts <<EOF
127.0.0.1 localhost localhost.localdomain
#{ETC_HOSTS_ENTRIES}
EOF

hostname --fqdn > /etc/hostname && hostname -F /etc/hostname
sed "s/^[ \t]*//" -i /etc/hosts

echo "SHELL PROVISIONING: Disabling selinux"
sed -i s/SELINUX=permissive/SELINUX=disabled/g /etc/selinux/config

SCRIPT

# hold script to update ansible master's known_hosts into a variable
$ANSIBLE_MASTER_SCRIPT = <<ANSIBLE
#!/bin/bash

echo "SHELL PROVISIONING: updating /home/vagrant/.ssh/known_hosts"
if [ -f /home/vagrant/.ssh/known_hosts ]; then
    rm -f /home/vagrant/.ssh/known_hosts
fi
#{KNOWN_HOSTS_ENTRIES}

ANSIBLE

Vagrant.configure(VAGRANT_API_VERSION) do |config|
    # disable box udpate on import
    config.vm.box_check_update = false

    # SHELL PROVISION
    config.vm.provision :shell, :inline => $DEFAULT_SHELL_SCRIPT

    # configure the hosts in cluster
    PONDERING_POND.each do |host|
        config.vm.define host[:hostname] do |host_config|
            host_config.vm.box = host[:box]
            host_config.vm.box_version = host[:version]
            host_config.vm.synced_folder "./ponder-ansible", "/vagrant", disabled: false
            host_config.vm.hostname = "#{host[:hostname]}.#{HOST_DOMAIN}"
            host_config.ssh.forward_agent = true
            host_config.hostmanager.aliases = "#{host[:hostname]}"
            host_config.vm.network "private_network", ip: host[:ip]
            if host[:forwarded_ports] then
                host[:forwarded_ports].each do |rule|
                    host_config.vm.network "forwarded_port", guest: rule[:guest], host: rule[:host]
                end
            end
            host_config.vm.provider :virtualbox do |vbox|
                vbox.name = host[:hostname].to_s
                vbox.gui = host[:gui]
                vbox.customize ["modifyvm", :id, "--memory", host[:ram].to_s ]
                vbox.customize ["modifyvm", :id, "--cpus", host[:cpu].to_s ]
                vbox.customize ["modifyvm", :id, "--ioapic", "on"]
                vbox.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
                vbox.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
            end

            # ansible provisioning for ansible master
            if host[:ansible_enabled] then
                # ensure that vm has entries of all vms in known_hosts file
                config.vm.provision :shell, :inline => $ANSIBLE_MASTER_SCRIPT
                host_config.vm.provision :ansible_local do |ansible|
                    # ansible inventory groups
                    ansible.groups = {
                        "ansible_master" => [
                            "ansible01"
                        ],
                        "server" => [
                            "server01"
                        ],
                        "client" =>[
                            "client01",
                            "client02"
                        ]
                    }
                    ansible.extra_vars = {
                      # any play/role related vars here
                      server_list: "{{ groups.server }}",
                      client_list: "{{ groups.client }}"
                    }
                    ansible.limit = "all,!ansible_master" # IMPORTANT
                    # append plays into same playbook or
                    # ansible.playbook = "plays/server-pond.yml"
                    # ansible.playbook = "plays/client-pond.yml"
                    ansible.playbook = "plays/all-pond.yml"
                    ansible.verbose = "-v"
                    compatibility_mode = "2.0"
                end
            end
        end
    end
    # vagrant-reload plugin required - vagrant plugin install vagrant-reload
    # below disables rebooting vm on re-running provisioning
    # config.vm.provision :reload
end
