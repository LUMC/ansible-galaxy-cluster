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


### Connect server to the cluster filesystem

- If required: add a few pre_tasks to mount the cluster filesystem. Example
shown below:

```YAML
    - name: Install nfs client.
      apt:
        name: nfs-common 
        state: present 
    - name: Mount /cluster/galaxy 
      mount: 
        src: hpc-cluster.example.org:/clusterfs/galaxy
        path: /cluster/galaxy
        fstype: nfs 
        opts: rw,hard,intr,rsize=32768,wsize=32768,tcp,vers=4,noatime
        dump: "0"
        passno : "0"
        state: mounted
```

- Set `galaxy_cluster_dir: /cluster/galaxy/mygalaxy` in `/inventory/host_vars/mygalaxy.yml`

### Set up HTTPS

- Create a server key and request a certificate. You can create a self-signed 
  certificate with `openssl req -new -newkey rsa:2048 -sha256 -days 3650 -nodes -x509 -keyout server.key -out server.crt`
- Set `galaxy_server_key` and `galaxy_server_cert` in the configuration.

### Configure the connection to the SLURM cluster


In this example we will be using SSH to connect to a login node. 
This way our server does not have to have the munge authentication needed
to connect to the cluster. The following steps were taken:

#### Set up user

- Request a service account user on your cluster. The user in this example
  is called `sa_galaxy`.
- Request a galaxy group on your cluster filesystem. (`gx_group` in this example)
- Get the UID for the service account user. (`12345` in this example)
- Get the GID for the Galaxy group. (`23456` in this example)
- Make sure the `galaxy_user` and `galaxy_group` variables are set in ansible:
```yaml
galaxy_user:
  name: sa_galaxy 
  uid: 12345
galaxy_group:
  name: gx_group
  uid: 23456
galaxy_create_user: true  
```

These steps are necessary in order to have the same users on the server as
on the cluster system.

#### Set up SSH keys:

- Add the following tasks to the playbook in order to create an SSH key for 
  the user:

```yaml
    - name: Set ssh directory
      set_fact:
        galaxy_ssh_directory: "/home/{{ galaxy_user }}/.ssh"
    - name: Create ssh directory for galaxy user
      file:
        state: directory
        mode: 0700
        path: "{{ galaxy_ssh_directory }}"
        owner: "{{ galaxy_user }}"

    - name: Generate keypair for galaxy user.
      community.crypto.openssh_keypair:
        path: "{{ galaxy_ssh_directory }}/id_rsa"
        owner: "{{ galaxy_user }}"
        mode: 0600
  
    - name: Read public key for galaxy user.
      ansible.builtin.slurp:
        src: "{{ galaxy_ssh_directory }}/id_rsa.pub"
      register: public_key

    - name: Show public key for galaxy user.
      ansible.builtin.debug:
        msg: "{{ public_key['content'] | b64decode }}"
```

- The public key should then be added to the authorized_keys file of the 
  user on the SLURM cluster.

#### Set up singularity.
- Add the following lines to the playbook tasks:
```
---
    - name: Install go
      import_role:
        name: gantsign.golang

    - name: Install singularity
      include_role:
        name: cyverse-ansible.singularity
      vars:
        singularity_go_path: "{{ golang_install_dir }}"
```
- Set the desired singularity version with `singularity_version`

#### Set up job_conf.xml

- create a `templates` folder.
- Create `templates/job_conf.xml.j2`:
```xml
<?xml version="1.0"?>
<job_conf>
    <plugins workers="4">
        <plugin id="local" type="runner" load="galaxy.jobs.runners.local:LocalJobRunner"/>
        <plugin id="cli" type="runner" load="galaxy.jobs.runners.cli:ShellJobRunner" />
    </plugins>
    <handlers>
        <handler id="handler0"/>
        <handler id="handler1"/>
    </handlers>
    <destinations default="dtd_destination">
        <destination id="local" runner="local"/>
        <destination id="dtd_destination" runner="dynamic">
            <param id="type">dtd</param>
        </destination>
        <expand macro="slurm_destination" id="slurm_short" ncpus="1" mem="2G" time="59" partition="short" />
        <expand macro="slurm_destination" id="slurm" ncpus="1" mem="4G" time="24:00:00" partition="all"/>
        <expand macro="slurm_destination" id="slurm_highmem" ncpus="1" mem="16G" time="24:00:00" partition="all,highmem" />
        <expand macro="slurm_destination" id="slurm_mid_highmem" ncpus="4" mem="64G" time="24:00:00" partition="all,highmem" />
        <expand macro="slurm_destination" id="slurm_high_highmem" ncpus="8" mem="112G" time="24:00:00" partition="all,highmem"/>
        <expand macro="slurm_destination" id="slurm_mid" ncpus="4" mem="16G" time="24:00:00" partition="all"/>    
    </destinations>    
    <macros>
        <xml name="slurm_destination" tokens="id,time,ncpus,mem,partition">
            <destination id="@ID@" tags="slurm_cluster" runner="cli">
                <env id="GALAXY_LIB">{{ galaxy_server_dir }}/lib</env>
                <env id="GALAXY_VIRTUAL_ENV">{{ galaxy_venv_dir }}</env>
                <param id="singularity_enabled">true</param>
                <param id="singularity_cmd">/cluster/software/singularity/3.7.3/bin/singularity</param>
                <!-- Isolate all things except environment. Galaxy passes settings 
                including PYTHONPATH in the environment. This is needed for galaxy
                settings to propagate to the tool script.
                Also set a current working directory to prevent the script from starting in $HOME.
                -->
                <param id="singularity_run_extra_arguments">--contain --ipc --pid --pwd $_GALAXY_JOB_DIR/working</param>
                <param id="shell_plugin">SecureShell</param>
                <param id="job_plugin">Slurm</param>
                <param id="shell_hostname">My_login_node</param>
                <param id="job_time">@TIME@</param>
                <param id="job_ncpus">@NCPUS@</param>
                <param id="job_partition">@PARTITION@</param>
                <param id="job_--mem">@MEM@</param>

                <param id="singularity_enabled">true</param>
                  <!-- Ensuring a consistent collation environment is good for reproducibility. -->
                <env id="LC_ALL">C</env>
                <!-- The cache directory holds the docker containers that get converted. -->
                <env id="SINGULARITY_CACHEDIR">/tmp/singularity</env>
                <!-- Singularity uses a temporary directory to build the squashfs filesystem. -->
                <env id="SINGULARITY_TMPDIR">/tmp</env>
            </destination>
        </xml>
    </macros>

    <limits>
        <limit type="registered_user_concurrent_jobs">50</limit>
        <limit type="anonymous_user_concurrent_jobs">0</limit>
        <limit type="destination_user_concurrent_jobs" id="local">2</limit>
        <limit type="destination_total_concurrent_jobs" id="local">2</limit>
        <limit type="destination_total_concurrent_jobs" tag="slurm_cluster">100</limit>
        <limit type="walltime">24:00:00</limit>
    </limits>

</job_conf>

```

- create `files/tool_destinations.yml` for example:
```YAML
default_destination: slurm
tools:
  upload1:
    default_destination: slurm_short
  ucsc_table_direct1:
    default_destination: slurm_short
  bowtie2:
    default_destination: slurm_highmem
  
  centrifuge:
    default_destination: slurm_highmem
  
  spades:
    default_destination: slurm_highmem

  rna_star:
    default_destination: slurm_mid_highmem
  
  rna_star_index_builder_data_manager:
    default_destination: slurm_mid_highmem
  
  kraken_database_builder:
    default_destination: slurm_high_highmem
  
verbose: True
```

- Create `templates/containers_resolvers_conf.xml.j2`
```xml
<containers_resolvers>
  <explicit_singularity />
  <cached_mulled_singularity cache_directory="{{ galaxy_mutable_data_dir }}/cache/singularity" />
  <mulled_singularity auto_install="False" cache_directory="{{ galaxy_mutable_data_dir }}/cache/singularity" />
  <build_mulled_singularity auto_install="False" cache_directory="{{ galaxy_mutable_data_dir }}/cache/singularity" />
</containers_resolvers>
```

- In your ansible settings set the following variables:

```YAML
galaxy_config_files:
  - src: tool_destinations.yml
    dest: "{{ galaxy_config_dir }}/tool_destinations.yml"
galaxy_config_templates:
  - src: job_conf.xml.j2
    dest: "{{ galaxy_config_dir }}/job_conf.xml"
  - src: container_resolvers_conf.xml.j2
    dest: "{{ galaxy_config_dir }}/container_resolvers_conf.xml"
```

- in ansible configuration set the following settings:

```YAML
galaxy_config:
  galaxy:
    # ... other settings can go here ...
    conda_auto_init: false
    conda_auto_install: false
    dependency_resolvers: {}
    job_config_file: "{{ galaxy_config_dir}}/job_conf.xml"
    tool_destinations_config_file: "{{ galaxy_config_dir}}/tool_destinations.yml"
    containers_resolvers_config_file: "{{ galaxy_config_dir }/container_resolvers_conf.xml"
```

### Further configuration

Further configuration of galaxy is detailed on the [ansible-galaxy README](
https://github.com/galaxyproject/ansible-galaxy/blob/main/README.md).

## Administration

### NGiNX
Nginx is 