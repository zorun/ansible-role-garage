# Ansible role for Garage

This Ansible role installs and configure [Garage](https://garagehq.deuxfleurs.fr/),
an open-source distributed object storage service tailored for self-hosting.

It downloads a binary release of Garage, creates a system user, setups data and
metadata directories, generates a configuration file, and finally installs a systemd
service to run Garage.

Currently, this role does not automatically connect nodes to each other,
but it is a planned feature.

## Installation

This role is available on [Ansible Galaxy](https://galaxy.ansible.com/zorun/garage)

## Basic configuration

The minimum required configuration is:

- a template for Garage's configuration file
- four variables: `garage_version`, `garage_local_template`, `garage_metadata_dir`, `garage_data_dir`

Here is an example playbook:

```
- hosts: mycluster
  roles:
    - garage
  vars:
    garage_version: "0.8.0"
    garage_local_template: "garage.toml.j2"
    garage_metadata_dir: "/var/lib/garage"
    garage_data_dir: "/mnt/data"
    my_rpc_secret: "130458bfce56b518db49e5f72029070b5e0fcbe514052c108036d361a087643f"
    my_admin_token: "7b3e91b552089363ab94eb95f62324fb4138c9a6d71a69daefae0c5047b33bb7"
```

You also need to create a file `templates/garage.toml.j2` at the root of your
Ansible directory with the following content:

```
# Managed by Ansible

metadata_dir = "{{ garage_metadata_dir }}"
data_dir = "{{ garage_data_dir }}"
db_engine = "lmdb"

replication_mode = "3"
block_size = 1048576
compression_level = 1

rpc_bind_addr = "{{ ansible_default_ipv4.address }}:3901"
rpc_public_addr = "{{ ansible_default_ipv4.address }}:3901"
rpc_secret = "{{ my_rpc_secret }}"

bootstrap_peers = []

[s3_api]
s3_region = "garage"
api_bind_addr = "[::]:3900"
root_domain = ".s3.garage.localhost"

[s3_web]
bind_addr = "[::]:3902"
root_domain = ".web.garage.localhost"
index = "index.html"

[admin]
api_bind_addr = "[::1]:3903"
admin_token = "{{ my_admin_token }}"
```

In this example, we use the main IPv4 address of each node as RPC address.
If your nodes are behind NAT, you will need to set `rpc_public_addr` to the public
IP address of each node.
If you are building an IPv6 cluster, you could use `{{ ansible_default_ipv6.address }}`.

In addition, this example uses two custom variables: `my_rpc_secret` and `my_admin_token`.
Feel free to use custom variables for any configuration entry you would like to manage.

Having to provide a template is cumbersome, but it gives a lot of flexibility.
Refer to the [official doc](https://garagehq.deuxfleurs.fr/documentation/reference-manual/configuration/)
to configure Garage according to your requirements.

## Variable reference

All role variables are listed below, followed by a short description.
A few of these variables are mandatory.  For all other variables, we indicate
the default value.

    garage_version (mandatory)

Version of Garage to download and use, without the initial `v`.  Example: `0.8.0`.

    garage_local_template (mandatory)

Local path to a template for Garage's configuration file.  When using
a relative path, see [Ansible's search path documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbook_pathing.html).

    garage_metadata_dir (mandatory)

Where metadata will be stored by Garage.  The role will create this directory
with appropriate permissions.

    garage_data_dir (mandatory)

Where actual data will be stored by Garage.  The role will create this directory
with appropriate permissions.

    garage_config_file: /etc/garage.toml

Location of the configuration file to generate on the target host.

    garage_systemd_service: garage

Name of the systemd service.  Useful if you plan to run several Garage
daemons on the same host.

    garage_binaries_dir: /usr/local/bin

Directory in which to store downloaded Garage binaries.  Each file in this
directory will be named with the version as suffix, e.g. `/usr/local/bin/garage-0.8.0`.

    garage_main_binary: /usr/local/bin/garage

Path to the main binary used by the systemd service.  This will be a symlink
to the requested version of Garage.

    garage_system_user: garage

Name of system user to create.  Garage will be run as this user, and
all files (both data and metadata) will belong to this user.

    garage_system_group: garage

Name of system group to create.

    garage_logging: netapp=info,garage=info

Logging configuration for Garage, provided through `RUST_LOG`.
See [env_logger documentation](https://docs.rs/env_logger/latest/env_logger/#enabling-logging)
for details on the syntax.

    garage_architecture: {{ansible_architecture}}

CPU architecture for the downloaded binary.  It should be set automatically
to the correct target host architecture, but you can still override it in case
it is wrong (e.g. if you want to run the `i686` binary).

## Advanced setup: multiple Garage daemons

Let's assume you want to run multiple Garage daemons on the same machine.
For example, you might have several clusters with some overlapping nodes.

Here is an example Ansible inventory:

```
[cluster1]
host1
host2
host3

[cluster2]
host1
host2
host3
host4
host5
```

You can manage this situation through the following playbook:

```
- hosts: cluster1
  roles:
    - garage
  vars:
    garage_version: "0.8.0"
    garage_local_template: "garage.toml.j2"
    garage_config_file: /etc/garage-cluster2.toml
    garage_metadata_dir: "/var/lib/garage/cluster1"
    garage_data_dir: "/mnt/data/cluster1"
    garage_systemd_service: garage-cluster1
    garage_main_binary: /usr/local/bin/garage-cluster1
    garage_system_user: garage-cluster1
    garage_system_group: garage-cluster1

- hosts: cluster2
  roles:
    - garage
  vars:
    garage_version: "0.8.1"
    garage_local_template: "garage.toml.j2"
    garage_metadata_dir: "/var/lib/garage/cluster2"
    garage_data_dir: "/mnt/data/cluster2"
    garage_config_file: /etc/garage-cluster2.toml
    garage_systemd_service: garage-cluster2
    garage_main_binary: /usr/local/bin/garage-cluster2
    garage_system_user: garage-cluster1
    garage_system_group: garage-cluster1
```

It is safe to share the same `garage_version` and `garage_local_template`.
You can also share the same user and group, depending on your threat model.

Note that if you want to use the garage CLI, you would need to run something
like `garage-cluster1 -c /etc/garage-cluster1.toml status`.  Similarly, to
restart the service: `systemctl restart garage-cluster1`.

## Upgrading a cluster

First, make sure to carefully read the [official upgrade documentation](https://garagehq.deuxfleurs.fr/documentation/cookbook/upgrading/)

### Straightforward upgrades

For a safe "straightforward upgrade":

- read [release notes](https://git.deuxfleurs.fr/Deuxfleurs/garage/releases)
- increase `garage_version`
- add `serial: 1` to your playbook (see [Ansible documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html#using-keywords-to-control-execution))
- run `ansible-playbook` with `--step`
- after each host has finished upgrading, check that everything works as expected, and tell Ansible to `(c)ontinue`

### Advanced upgrades

For upgrades between incompatible versions, the exact strategy depends on
the version: please refer to the official documentation and specific migration guide.

If you need to download the new version of Garage without touching the existing
setup, you can run the "download" task alone and force the version:

```
ansible-playbook garage.yaml -e garage_version=0.9.0 --tags download
```

You should now have the new binary on your hosts, accessible through its
full path (e.g. `/usr/local/bin/garage-0.9.0`).  This allows you to run
offline migrations.
