# Setting up galaxy on a cluster.

## Introduction

Galaxy is a fairly complex server application with a lot of moving parts. 
Setting up Galaxy in production is complicated, and making sure jobs are 
executed on an external cluster even more so. In this tutorial you will set up
a production Galaxy connected to a SLURM cluster where the jobs run in 
singularity.

## Requirements

### Server

- OS: Ubuntu 20.04 or later
- Connected to the HPC filesystem
- Have access to the HPC scheduler. This can be via commands (sbatch for example) 
  but in this example we use a service account (sa_galaxy) and submit our jobs
  over ssh.

### Software
- ansible 2.10. Can be installed with `pip install ansible~=2.10`.


## Setting up galaxy

### Preparation
- Create a directory where your configuration where your playbook will live
    - `mkdir -p mygalaxy`
- Install ansible role requirements: `ansible-galaxy install -r role-requirements.yml`. 
  The role-requirements file can be found [here](example/role-requirements.yml).

### Setting up a playbook

We will be working in the `mygalaxy` directory that was created in the previous
step. 
- Create a hosts file. `mkdir inventory && touch inventory/hosts`
- Edit the hosts file:

```
[cluster-galaxy]
mygalaxy host=mygalaxy.example.org
testgalaxy host=testgalaxy.example.org 

[test]
testgalaxy host=testgalaxy.org

[production]
mygalaxy host=mygalaxy.example.org
```

- Create a group settings file: 
  `mkdir inventory/group_vars && touch inventory/group-vars/cluster-galaxy.yml`
  In this file you can set a shared configuration for your galaxy and your 
  testgalaxy.

- Create settings files for individual galaxies
    - `mkdir inventory/host_vars`
    - touch inventory/host_vars/mygalaxy.yml
    - touch inventory/host_vars/testgalaxy.yml

- create a basic playbook `playbook.yml`:

```YAML
- name: Galaxy server 
  hosts: all
  gather_facts: true 
  become: true 

  pre_tasks:
    - name: Install go
      import_role:
        name: gantsign.golang

    - name: Install singularity
      include_role:
        name: cyverse-ansible.singularity
      vars:
        singularity_go_path: "{{ golang_install_dir }}"
  roles:
    - role: lumc.galaxy-cluster
```

- This is a good time to initialize a git repository with `git init` and 
  commit the directory. Git allows you to rollback to each commit as well as
  providing an easy way to backup your configuration in a remote location.
