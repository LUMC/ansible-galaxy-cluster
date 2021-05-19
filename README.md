ansible-galaxy-cluster
======================

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

Requirements
------------

The [galaxyproject.galaxy role](https://github.com/galaxyproject/ansible-galaxy) 
should be installed and available for ansible.

Role Variables
--------------

Important variables 
...................

Variable| default | description
---|---|---
`galaxy_cluster_dir` | /cluster/galaxy | Location on the cluster where galaxy should be installed.
`galaxy_python_version` | "3.7" | Which python version should be used to install galaxy
`galaxy_server_key` | *REQUIRED* | The SSL server key.
`galaxy_server_cert | *REQUIRED* | The SSL server certificate


`galaxy_db_username`| galaxy | Username for the galaxy database
`galaxy_db_name` | galaxy | database 
A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

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
