postgresql-pacemaker-example-playbook [![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/hyperium/hyper/master/LICENSE)
=====================================

An Ansible example playbook for set up PostgreSQL active standby cluster using Pacemaker on LXD containers.

This is just an example and have the following limitations:

* STONITH is not used.
* quorum node is not used.

That is, there is no care for split brains situations. In production environment, you should care those.

## Prerequisite

Setup LXD.

## Set up

### Set up virtualenv and install the latest ansible which has the LXD module

```
virtualenv venv
source venv/bin/activate
pip install git+https://github.com/ansible/ansible
```

### Modify lxd-bridge DHCP range and restart lxd-bridge

To leave the space for static IP addresses, edit `/etc/default/lxd-bridge`.
Choose the IP addresses accordingly to the value of `LXD_IPV4_NETWORK`.

For example:

```
## IPv4 network (e.g. 10.0.8.0/24)
LXD_IPV4_NETWORK="10.155.92.1/24"

## IPv4 DHCP range (e.g. 10.0.8.2,10.0.8.254)
LXD_IPV4_DHCP_RANGE="10.155.92.200,10.155.92.254"
```


Then restart lxd-bridge.

```
sudo systemctl restart lxd-bridge
```

### Edit static IP addresses in variable file

Edit IP addresses in `group_vars/development/vars.yml` appropriately.

### Optionally edit passwords and ssh private key and public key

Passwords and ssh keys are stored in `group_vars/development/secrets.yml`.

If you want to change the values in this file, do the following steps.

Decrypt the file.

```
ansible-vault decrypt group_vars/development/secrets.yml
```

You will be asked the vault password.
The initial password is `password` which is OK since this playbook is just an example.
In a real use case, you must choose a strong password.

Edit the values in `group_vars/development/secrets.yml`.

Encrypt the file again.

```
ansible-vault encrypt group_vars/development/secrets.yml
```

You will be asked to set the new vault password.


### Run playbooks

#### Create and launch containers

```
ansible-playbook launch_containers.yml -D -v
```

You will be asked the vault password.
