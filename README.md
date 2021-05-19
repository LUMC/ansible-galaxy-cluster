# ansible-galaxy-cluster

This role will set up [Galaxy](https://galaxyproject.org) on a server connected to
the cluster filesystem. It does the following:

- Makes sure all necessary components of galaxy are available on the cluster filesystem.
- Installs python on the cluster filesystem (using conda) and utilizes this to
  create the galaxy virtual environment. This ensures all nodes on the cluster
  utilize the same python binary for tasks that require the galaxy virtualenv.
- uses the [galaxyproject.galaxy role](https://github.com/galaxyproject/ansible-galaxy)
  to install galaxy. Sets all the defaults necessary.
- Installs docker and sets up postgres and nginx on the server using docker.
- Sets up galaxy as a systemd service
- Sets up a systemd timer service that regularly makes a SQL dump of the database.


Not included:
- configuring job_conf.xml, checkout the [galaxyproject.galaxy role](https://github.com/galaxyproject/ansible-galaxy) for documentation on how to add configuration files. 
Checkout the [advanced job_conf example](https://github.com/galaxyproject/galaxy/blob/dev/lib/galaxy/config/sample/job_conf.xml.sample_advanced)
for information on how to setup galaxy for your cluster.

For an example check [HOWTO.md](HOWTO.md).

## Requirements

The [galaxyproject.galaxy role](https://github.com/galaxyproject/ansible-galaxy) 
should be installed and available for ansible.

## Role Variables

### Important variables 

Variable| default | description
---|---|---
`galaxy_cluster_dir` | /cluster/galaxy | Location on the cluster where galaxy should be installed.
`galaxy_python_version` | "3.7" | Which python version should be used to install galaxy
`galaxy_server_key` | *REQUIRED* | The SSL server key.
`galaxy_server_cert` | *REQUIRED* | The SSL server certificate


### Other variables

Variable| default | description
---|---|---
`galaxy_db_username`| galaxy | Username for the galaxy database
`galaxy_db_name` | galaxy | The name of the galaxy database
`galaxy_minicoda_download_url` | https://repo.anaconda.com/miniconda/Miniconda3-py37_4.9.2-Linux-x86_64.sh | The default download url for miniconda.
`galaxy_miniconda_download_location` | /tmp/miniconda.sh | Where to store the conda install script.
`nginx_dhparam_bits | 4096 | Nginx is more secure when a dhparam file is generated. Set the amount of bits for the generation.
`nginx_ssl_dir` | /etc/nginx/ssl |  where the dhparam files are stored
`nginx_config_dir | /etc/nginx | Configuration directory for nginx


Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
