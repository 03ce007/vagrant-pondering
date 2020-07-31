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
CONSUL_SERVER="consul-server0"
CONSUL_CLIENT="consul-client0"
DNSMASQ_SERVER="dnsmasq-server0"


PONDERING_POND = [
    # NOTE:
    #   1 - ensure to check below flags in each entry
    #       - ansible_enabled (bool) is set to true for ANSIBLE_MASTER_VM only
    #       - forwarded_ports (key/value) = exposing ports, if needed
    # an example
    { :hostname => "#{CONSUL_SERVER}1", :ip => "10.22.22.101", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false, :update_hosts_file => false },
    { :hostname => "#{CONSUL_SERVER}2", :ip => "10.22.22.102", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false, :update_hosts_file => false },
    { :hostname => "#{CONSUL_SERVER}3", :ip => "10.22.22.103", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false, :update_hosts_file => false, :forwarded_ports => [{ :guest => 8500, :host => 8500 }] },
    { :hostname => "#{CONSUL_CLIENT}1", :ip => "10.11.22.201", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false, :update_hosts_file => false },
    { :hostname => "#{CONSUL_CLIENT}2", :ip => "10.11.22.202", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false, :update_hosts_file => false },
    { :hostname => "#{DNSMASQ_SERVER}1", :ip => "10.33.44.100", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => false, :update_hosts_file => false },
    # ensure that ansible master is the last one
    { :hostname => "#{ANSIBLE_MASTER_VM}1", :ip => "10.99.1.100", :box => "#{DEFAULT_BOX}", :version =>"#{DEFAULT_BOX_VERSION}", :ram => "#{HOST_MEM_LOW_VAL}", :cpu =>"#{HOST_CPU_LOW_VAL}", :gui => false, :ansible_enabled => true, :update_hosts_file => false, :forwarded_ports => [] },
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
    KNOWN_HOSTS_ENTRIES << "ssh-keyscan #{n[:hostname]}.#{HOST_DOMAIN},#{n[:ip]} | grep ecdsa-sha2-nistp256 >> /home/vagrant/.ssh/known_hosts\n"
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

echo "SHELL PROVISIONING: creating directories"
mkdir -p /machines

echo "SHELL PROVISIONING: installing pip ..."
yum -y -q -e 0 install epel-release && yum -y -q -e 0 install python-pip && pip install --upgrade pip && pip install --upgrade setuptools

echo "SHELL PROVISIONING: updating /home/vagrant/.ssh/known_hosts"
if [ -f /home/vagrant/.ssh/known_hosts ]; then
    rm -f /home/vagrant/.ssh/known_hosts
fi

#{KNOWN_HOSTS_ENTRIES}

ANSIBLE

Vagrant.configure(VAGRANT_API_VERSION) do |config|
    # disable box udpate on import
    config.vm.box_check_update = false
    # hostmanager config
    # must have `vagrant-hostmanager` plugin installed
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = false
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = true

    # configure the hosts in cluster
    PONDERING_POND.each do |host|
        config.vm.define host[:hostname] do |host_config|
            host_config.vm.box = host[:box]
            host_config.vm.box_version = host[:version]
            host_config.vm.synced_folder ".", "/vagrant", disabled: true
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

            # update /etc/hosts if required
            if host[:update_hosts_file] then
                host_config.vm.provision :shell, :inline => $DEFAULT_SHELL_SCRIPT
            end

            # ansible provisioning for ansible master
            if host[:ansible_enabled] then
                # ensure that vm has entries of all vms in known_hosts file
                host_config.vm.provision :shell, :inline => $ANSIBLE_MASTER_SCRIPT
                # ensure that correct folders are mounted
                host_config.vm.synced_folder "./.vagrant/machines", "/machines", disabled: false
                host_config.vm.synced_folder "./ponder-ansible", "/vagrant", disabled: false
                # install ansible locally and run playbook
                host_config.vm.provision :ansible_local do |ansible|
                    ansible.host_vars = {
                        "consul-server01" => { "consul_node_role" => "bootstrap" }
                    }
                    ansible.extra_vars = {
                        ansible_ssh_user: "vagrant",
                        ansible_ssh_private_key_file: "/machines/{{ inventory_hostname }}/virtualbox/private_key"
                    }
                    ansible.galaxy_role_file="requirements.yml"
                    ansible.galaxy_roles_path="roles"
                    ansible.inventory_path = "inventories/pondering_pond"
                    ansible.limit = "all,!ansible_master" # IMPORTANT
                    # append plays into same playbook or
                    ansible.playbook = "plays/consul-pond.yml"
                    ansible.verbose = "-vv"
                    compatibility_mode = "2.0"
                end
            end
        end
    end
    # vagrant-reload plugin required - vagrant plugin install vagrant-reload
    # below disables rebooting vm on re-running provisioning
    # config.vm.provision :reload
end
