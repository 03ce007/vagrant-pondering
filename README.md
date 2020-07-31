# vagrant-pondering (work-in-progress)

Lets one ponder on the various tools and provides way of developing skills in them.

# What is in the pond?

  - consul
  - dnsmasq
  - apache-hue build



### Pre-requisites

```sh
$ vagrant plugin install vagrant-hosts
$ vagrant plugin install vagrant-hostmanager
$ vagrant box add bento/centos-7.7 --insecure
```
### Important

 - Ensure that `PONDERING_POND` dict in `Vagrantfile` matches with ansible inventory file `./ponder-ansible/inventories/pondering_pond`, most importantly the hostnames and ips must match in both files.

### Start pondering

```sh
$ vagrant up --no-provision # once ran, check all hosts have come up
$ vagrant up --provision    # provision with ansible
$ vagrant up ansible01 --provision # re-run playbook
```

#### Inventory

```sh
# ./ponder-ansible/inventories/pondering_pond
ansible01 ansible_connection=local
dnsmasq-server01 ansible_host=10.33.44.100
consul-server01 ansible_host=10.22.22.101 consul_node_role=bootstrap
consul-server02 ansible_host=10.22.22.102 consul_node_role=server
consul-server03 ansible_host=10.22.22.103 consul_node_role=server
consul-client01 ansible_host=10.11.22.201 consul_node_role=client
consul-client02 ansible_host=10.11.22.202 consul_node_role=client

[ansible_master]
ansible01

[dnsmasq_server]
dnsmasq-server01

[consul_server]
consul-server01
consul-server03
consul-server02

[consul_client]
consul-client01
consul-client02

[consul_instances:children]
consul_server
consul_client
```

### know issue

 - Consul: v1.8.0 fails first time when `config.json` is generated, but re-running goes in ok. Though v1.7.0 runs without this issue (may be because external role does not support v1.8.0 yet - https://github.com/ansible-community/ansible-consul )
 - Since `ansible.galaxy_role*` are defined in `Vagrantfile`, it downloads the roles everytime `--provision` is ran, making it difficult to disable/edit a task in external role (Tip: in such cases, let `--provision` download it once, and thenn disable `ansible.galaxy_role*` in `Vagrantfile`).