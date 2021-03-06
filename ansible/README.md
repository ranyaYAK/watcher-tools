# Watcher Ansible playbooks

This project proposes a large set of [Ansible](http://www.ansible.com/) 
playbooks you can use to easily deploy the 
[Watcher](https://wiki.openstack.org/wiki/Watcher)  and the 
[Watcher Metering](https://factory.b-com.com/www/watcher/doc/watcher-metering) 
chain  solutions.

## Getting Started

### Prerequisites

It’s expected that your system already has [python2.7](http://www.python.org/) 
and latest version of 
[pip](http://www.pip-installer.org/en/latest/installing.html)

### Install Ansible

Ansible package is available on PyPI repository. To install Ansible on your 
system :

``` shell
$ sudo pip install ansible
```

Ansible modules and its dependencies will be automatically installed on your 
system.

### Generate a SSH key pair for Ansible
Ansible communicates with remote machines 
[over SSH](http://docs.ansible.com/ansible/intro_getting_started.html#remote-connection-information). 
You have to create a [SSH key pair](https://help.ubuntu.com/community/SSH/OpenSSH/Keys) 
on your Ansible machine:

``` shell
$ ssh-keygen -t rsa -f ~/.ssh/id_rsa_ansible -N "" -C "Ansible"
```

Before you can use Ansible to deploy some software to your target machine you 
must send the previously generated public key to this machine.

``` shell
$ ssh-copy-id -i ~/.ssh/id_rsa_ansible.pub <username>@<your_target_machine>
```
This command will copy your public SSH key into the file 
`$HOME/.ssh/authorized_keys` on your target machine.

### Run Ansible
To validate SSH configuration, we can just ask to Ansible to run a quick command 
on target host.

Edit (or create) the [inventory](http://docs.ansible.com/ansible/intro_inventory.html) 
file `/etc/ansible/hosts` and put one or more remote systems in it (you can 
group target machines by group name)

``` shell
[controllers]
controller1 ansible_ssh_host=192.168.174.183 ansible_ssh_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible
controller2 ansible_ssh_host=openstack.controler.mylab.com ansible_ssh_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible
controller3 ansible_ssh_host=devstack_pc ansible_ssh_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible

[compute_nodes]
compute1 ansible_ssh_host=192.168.174.184 ansible_ssh_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible
#compute2 ansible_ssh_host=openstack.compute1.mylab.com ansible_ssh_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible
compute3 ansible_ssh_host=devstack_pc ansible_ssh_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible

```

and run Ansible to ping all your target machines:


``` shell
$ ansible all -m ping
```

## Defined Ansible roles for Watcher
Different [roles](http://docs.ansible.com/ansible/playbooks_roles.html) have 
been created to easily customize Watcher deployment on target machines:

- `nanoconfig-server`: Deploy the [nanoconfig](https://github.com/nanomsg/nanoconfig) 
server, used for the Watcher Metering chain
- `service-discovery-agent`: Deploy a Service Discovery Agent 
([registrator](https://github.com/gliderlabs/registrator) and 
[consul](https://github.com/hashicorp/consul) agent mode), used for the Watcher 
Metering chain
- `service-discovery-cluster`: Deploy a Service Discovery cluster (registrator 
and consul cluster mode), used for the Watcher Metering chain
- `watcher-metering-agent`: Deploy a Watcher Metering Agent
- `watcher-metering-engine`: Deploy Watcher Metering third components 
([Influxdb](https://influxdb.com/), [Riemann](http://riemann.io/) and 
[Grafana](http://grafana.org/))
- `watcher-metering-publisher`: Deploy a Watcher Metering Publisher
- `watcher`: Deploy the OpenStack Watcher module.

## Defined Ansible Tags for Watcher:
You can use [tags](http://docs.ansible.com/ansible/playbooks_tags.html) to only 
run part of the `playbook.yml` playbook.Tags are:
- `service-discovery`
- `watcher-metering-engine-publisher` 
- `watcher-metering-publisher` 
- `metering-agent`
- `metering-publisher`
- `metering-engine`
- `compute-node`
- `watcher-controller`

Here an example of using tags with ansible:

``` shell
$ ansible-playbook -s playbook.yml -i production --ask-sudo-pass --tags=compute-node
```

According to the playbook, this command will deploy the 
[Watcher Metering Agent](https://github.com/b-com/watcher-metering) on all target 
machines referenced in the hosts group named `compute-node` (defined in the file 
`/etc/ansible/hosts`)

``` shell
- hosts: compute-node
  become: yes
  become_user: root
  tags:
    - compute-node
    - metering-agent
  roles:
    - ansible-pip
    - watcher-metering-agent
```


Deploy Watcher Metering Chain for development
=============================================

For development, you can bypass the service discovery facilities. But you will have to manually set networking parameters. 

You can manually install [watcher](https://factory.b-com.com/www/watcher/doc/watcher/deploy/installation.html) and a [watcher metering agent](https://factory.b-com.com/www/watcher/doc/watcher-metering/deploy/installation.html) into DevStack and deploy, with Ansible, the watcher Metering chain into another machine.

Declare in the file `production` inventory file, group name `all-in-one`, your target machine (ex: 192.168.174.183) on which you want deploy Watcher Metering components:

 ``` shell
[all-in-one]
my_machine ansible_ssh_host=192.168.174.183 ansible_ssh_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible
```

Edit the `./group_vars/all.yml` file and update the `target_allinone_host` key with your machine IP address:
``` shell
target_allinone_host: "192.168.174.183"
```

And run ansible:
``` shell
$ ansible-playbook -s playbook.yml -i production --ask-sudo-pass --limit=all-in-one
```

This command will run this part of playbook:
``` shell
- hosts: all-in-one
  become: yes
  become_user: root
  roles:
    - docker
    - docker-compose
    - watcher-metering-engine
    - watcher-metering-publisher
``` 

Finally, you have to configure the Watcher Metering Agent, deployed into DevStack, by setting its Publisher endpoint URL:
``` shell
# Complete Publisher API endpoint URI, e.g. tcp://localhost:12345.
# Used if the `use_nanoconfig_service` option is not activated.
# (string value)
publisher_endpoint = 192.168.174.183:12345
```

and restart the agent.


If your installation is successful, you will have access to :
* InfluxDB Web Interface : http://192.168.174.183:8083
* Grafana Web Interface: http://192.168.174.183:3000 (you should see metering data into `Watcher metering I&T` dashboard)
 
