# Hands-On: Provision a Docker Swarm Cluster with Vagrant and Ansible

### Automatically provision a Docker Swarm cluster composed of three masters and two workers

## Preconditions:

* Linux, Windows or macOS host with at least 8GB RAM
* VirtualBox - https://www.virtualbox.org/
* Vagrant - https://www.vagrantup.com/

## Code Overview

Through the Vagrant script (_Vagrantfile_) we will provision five identical Linux machines (Ubuntu 18.04 LTS) on the same subnet:

```ruby
nodes = [
  { :hostname => 'swarm-master-1', :ip => '192.168.77.10', :ram => 1024, :cpus => 1 },
  { :hostname => 'swarm-master-2', :ip => '192.168.77.11', :ram => 1024, :cpus => 1 },
  { :hostname => 'swarm-master-2', :ip => '192.168.77.12', :ram => 1024, :cpus => 1 },  
  { :hostname => 'swarm-worker-1', :ip => '192.168.77.13', :ram => 1024, :cpus => 1 },
  { :hostname => 'swarm-worker-2', :ip => '192.168.77.14', :ram => 1024, :cpus => 1 }
]

Vagrant.configure("2") do |config|
  # Always use Vagrant's insecure key
  config.ssh.insert_key = false
  # Forward ssh agent to easily ssh into the different machines
  config.ssh.forward_agent = true
  # Provision nodes
  nodes.each do |node|
    config.vm.define node[:hostname] do |nodeconfig|
      nodeconfig.vm.box = "bento/ubuntu-18.04";
      nodeconfig.vm.hostname = node[:hostname] + ".box"
      nodeconfig.vm.network :private_network, ip: node[:ip]
      memory = node[:ram] ? node[:ram] : 1024;
      cpus = node[:cpus] ? node[:cpus] : 1;
      nodeconfig.vm.provider :virtualbox do |vb|
        vb.customize [
          "modifyvm", :id,
          "--memory", memory.to_s,
          "--cpus", cpus.to_s
        ]
      end
    end
  end
  # In addition, swarm-worker-2 is the Ansible server
  config.vm.define "swarm-worker-2" do |ansible|
    # Provision Ansible playbook
    ansible.vm.provision "file", source: "../Ansible", destination: "$HOME"
    # Install Ansible and configure nodes
    ansible.vm.provision "shell", path: "ansible.sh"
  end
end
```

A Bash script is then injected in node _swarm-worker-2_ to install Ansible via _pip_:

```sh
$ sudo apt-get install -y python3-pip sshpass
$ sudo -H pip3 install --upgrade pip
$ sudo -H pip3 install ansible
```

The cluster configuration is carried out by means of three Ansible playbooks. The first one, _cluster.yaml_, installs Docker CE on all four hosts, regardless of their target role:

```yaml
---
- hosts: all
  become: true
  vars_files:
  - vars.yml
  strategy: free

  tasks:
    - name: Add the docker signing key
      apt_key: url=https://download.docker.com/linux/ubuntu/gpg state=present

    - name: Add the docker apt repo
      apt_repository: repo='deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable' state=present

    - name: Install packages
      apt:
        name: "{{ PACKAGES }}"
        state: present
        update_cache: true
        force: yes

    - name: Add user vagrant to the docker group
      user:
        name: vagrant
        groups: docker
        append: yes
```

The second playbook, _master.yaml_, initializes the Docker Swarm, and configures the host named _swarm-master-1_ as the (leader) Swarm Manager.

```yaml
---
- hosts: swarm-master-1
  become: true

  tasks:
    - name: Initialize the cluster
      shell: docker swarm init --advertise-addr 192.168.77.10 >> cluster_initialized.txt
      args:
        chdir: $HOME
        creates: cluster_initialized.txt
```

The last playbook, _join.yaml_, sets up the actual Docker Swarm as composed of four hosts - two masters (so that the manager role is redunded) and two workers. In order to achieve that, two _join-tokens_ (one to join the cluster as a manager, and one to join the cluster as a worker) are generated on the host which initialized the swarm, and automatically passed to the remaining three hosts.

```yaml
---
- hosts: swarm-master-1
  become: true
  gather_facts: false

  tasks:
    - name: Get master join command
      shell: docker swarm join-token manager
      register: master_join_command_raw

    - name: Set master join command
      set_fact:
        master_join_command: "{{ master_join_command_raw.stdout_lines[2] }}"

    - name: Get worker join command
      shell: docker swarm join-token worker
      register: worker_join_command_raw

    - name: Set worker join command
      set_fact:
        worker_join_command: "{{ worker_join_command_raw.stdout_lines[2] }}"

- hosts: swarm-master-2
  become: true

  tasks:
    - name: Master joins cluster
      shell: "{{ hostvars['swarm-master-1'].master_join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt

- hosts: workers
  become: true

  tasks:
    - name: Workers join cluster
      shell: "{{ hostvars['swarm-master-1'].worker_join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
```

## Step 1: Run the Vagrantfile

Type the following commands:

```sh
$ cd Vagrant
$ vagrant up
```

This command executes the _Vagrantfile_, which in turn - as esplained above - installs Ansible and runs the playbooks. All this will take a few minutes. Eventually, our Docker Swarm Cluster will be set up and ready to be used.

## Step 2: Verify the Swarm Cluster

Log in the first node using Vagrant's built-in SSH access as the _vagrant_ user:

```sh
$ vagrant ssh swarm-master-1
```

On this node, which acts as a Swarm Manager, all nodes are now shown in the inventory:

```sh
$ vagrant@swarm-master-1:~$ docker node ls
```
