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
$ ansible-playbook launch_containers.yml -D -v
```

You will be asked the vault password.

#### Set up a PostgreSQL cluster using Pacemaker in containers

```
$ ansible-playbook setup_containers.yml -D -v
```

In this example, we also print the date when the set up is complete.

```
$ ansible-playbook setup_containers.yml -D -v; date -u
...
Sun Aug 21 13:51:21 UTC 2016
```

### Fail over by stopping master container forcefully

#### Monitor PostgreSQL cluster state

Enter the standby container.

```
lxc exec node2 bash
```

Run the following command to monitor cluster state. Keep this terminal open.

```
[root@node2 ~]# crm_mon -fA
```

Wait until the cluster becomes the following state. `node1` has the master IP address and `node1` is the PostgreSQL master 
nd `node2` is the PostgreSQL standby.

```
Last updated: Sun Aug 21 13:52:07 2016          Last change: Sun Aug 21 13:52:03 2016 by root via crm_attribute on node1
Stack: corosync
Current DC: node1 (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
2 nodes and 3 resources configured

Online: [ node1 node2 ]

 Master/Slave Set: pgsql-master [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
master-ip       (ocf::heartbeat:IPaddr2):       Started node1

Node Attributes:
* Node node1:
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 0000000003000098
    + pgsql-status                      : PRI
    + pgsql-xlog-loc                    : 0000000003000098
* Node node2:
    + master-pgsql                      : -INFINITY
    + pgsql-data-status                 : STREAMING|ASYNC
    + pgsql-status                      : HS:async
    + pgsql-xlog-loc                    : 0000000003000000

Migration Summary:
* Node node2:
* Node node1:
```

#### Stop the master (node1) container

Run the following command on the LXD host to stop the master container forcefully.

```
$ lxc stop -f node1; date -u
Sun Aug 21 13:52:57 UTC 2016
```

After some time, you'll see that node2 has the master-ip and
PostgreSQL on node2 has been promoted to master.

```
Last updated: Sun Aug 21 13:53:11 2016          Last change: Sun Aug 21 13:53:05 2016 by root via crm_attribute on node2
Stack: corosync
Current DC: node2 (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
2 nodes and 3 resources configured

Online: [ node2 ]
OFFLINE: [ node1 ]

 Master/Slave Set: pgsql-master [pgsql]
     Masters: [ node2 ]
master-ip       (ocf::heartbeat:IPaddr2):       Started node2

Node Attributes:
* Node node2:
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 00000000030001A8
    + pgsql-status                      : PRI
    + pgsql-xlog-loc                    : 0000000003000000

Migration Summary:
* Node node2:
```

#### Start node1 again

Run the following command on the LXD host to start `node1` container.

```
$ lxc start node1; date -u
Sun Aug 21 13:53:58 UTC 2016
```

After 5 or seconds, you'll see `node1` is still offline.

```
Last updated: Sun Aug 21 13:53:11 2016          Last change: Sun Aug 21 13:53:05 2016 by root via crm_attribute on node2
Stack: corosync
Current DC: node2 (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
2 nodes and 3 resources configured

Online: [ node2 ]
OFFLINE: [ node1 ]

 Master/Slave Set: pgsql-master [pgsql]
     Masters: [ node2 ]
master-ip       (ocf::heartbeat:IPaddr2):       Started node2

Node Attributes:
* Node node2:
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 00000000030001A8
    + pgsql-status                      : PRI
    + pgsql-xlog-loc                    : 0000000003000000

Migration Summary:
* Node node2:
```

#### Let node1 join the cluster

Run the following command and enter `node1` container.

```
$ lxc exec node1 bash
```

In the real situation, you would investigate why the node had shutdown.
Since this is an example, skip that and remove the PostgreSQL lock file created by Pacemaker.

```
[root@node1 ~]# ll /var/run/postgresql/
total 4
-rw-r----- 1 root     root      0 Aug 21 13:52 PGSQL.lock
-rw-r----- 1 postgres postgres 36 Aug 21 13:52 rep_mode.conf
[root@node1 ~]# rm /var/run/postgresql/PGSQL.lock
rm: remove regular empty file '/var/run/postgresql/PGSQL.lock'? y
```

Then let `node1` join the cluster again.

```
[root@node1 ~]# pcs cluster start node1; date -u
node1: Starting Cluster...
Sun Aug 21 13:55:30 UTC 2016
```

After 15 seconds, you'll see that PostgreSQL on `node1` become standby mode.

```
Last updated: Sun Aug 21 13:55:45 2016          Last change: Sun Aug 21 13:55:42 2016 by root via crm_attribute on node2
Stack: corosync
Current DC: node2 (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
2 nodes and 3 resources configured

Online: [ node1 node2 ]

 Master/Slave Set: pgsql-master [pgsql]
     Masters: [ node2 ]
     Slaves: [ node1 ]
master-ip       (ocf::heartbeat:IPaddr2):       Started node2

Node Attributes:
* Node node1:
    + master-pgsql                      : 100
    + pgsql-data-status                 : STREAMING|SYNC
    + pgsql-status                      : HS:sync
* Node node2:
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 00000000030001A8
    + pgsql-status                      : PRI
    + pgsql-xlog-loc                    : 0000000003000000

Migration Summary:
* Node node2:
* Node node1:
```

Now stop monitoring on `node2` by pressing Control-C.

### Fail over by killing master PostgreSQL process forcefully

#### Start monitoring cluster state on node1

Run the fowllowing command on `node1` and keep this terminal open.

```
[root@node1 ~]# crm_mon -fA
```

You'll the output like the following:

```
Last updated: Sun Aug 21 13:57:17 2016          Last change: Sun Aug 21 13:55:42 2016 by root via crm_attribute on node2
Stack: corosync
Current DC: node2 (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
2 nodes and 3 resources configured

Online: [ node1 node2 ]

 Master/Slave Set: pgsql-master [pgsql]
     Masters: [ node2 ]
     Slaves: [ node1 ]
master-ip       (ocf::heartbeat:IPaddr2):       Started node2

Node Attributes:
* Node node1:
    + master-pgsql                      : 100
    + pgsql-data-status                 : STREAMING|SYNC
    + pgsql-status                      : HS:sync
* Node node2:
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 00000000030001A8
    + pgsql-status                      : PRI
    + pgsql-xlog-loc                    : 0000000003000000

Migration Summary:
* Node node2:
* Node node1:
```

#### Kill postgres process on node2

Run the following command on node2 to kill the postgres processes.

```
[root@node2 ~]# kill -KILL `head -1 /var/lib/pgsql/9.5/data/postmaster.pid`; date -u
Sun Aug 21 13:58:20 UTC 2016
```

After 11 seconds, you'll see `node1` is promoted to master.

```
Last updated: Sun Aug 21 13:58:31 2016          Last change: Sun Aug 21 13:58:27 2016 by root via crm_attribute on node1
Stack: corosync
Current DC: node2 (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
2 nodes and 3 resources configured

Online: [ node1 node2 ]

 Master/Slave Set: pgsql-master [pgsql]
     Masters: [ node1 ]
master-ip       (ocf::heartbeat:IPaddr2):       Started node1

Node Attributes:
* Node node1:
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 0000000003000398
    + pgsql-status                      : PRI
* Node node2:
    + master-pgsql                      : -INFINITY
    + pgsql-data-status                 : DISCONNECT
    + pgsql-status                      : STOP

Migration Summary:
* Node node2:
   pgsql: migration-threshold=2 fail-count=1000000 last-failure='Sun Aug 21 13:58:23 2016'
* Node node1:

Failed Actions:
* pgsql_start_0 on node2 'unknown error' (1): call=23, status=complete, exitreason='My data may be inconsistent. You have to remove /va
r/run/postgresql/PGSQL.lock file to force start.',
    last-rc-change='Sun Aug 21 13:58:23 2016', queued=0ms, exec=383ms
```

#### Let PostgreSQL on node2 be standby

Again in the real situation, you would investigate why the PostgreSQL process was killed.
Since this is an example, skip that and remove the PostgreSQL lock file created by Pacemaker.

```
[root@node2 ~]# ll /var/run/postgresql/
total 4
-rw-r----- 1 root     root      0 Aug 21 13:53 PGSQL.lock
-rw-r----- 1 postgres postgres 31 Aug 21 13:58 rep_mode.conf
[root@node2 ~]# \rm /var/run/postgresql/PGSQL.lock
```

Then run the following command to let the pgsql resource of Pacemaker on `node2` start again.

```
[root@node2 ~]# pcs resource failcount reset pgsql node2; date -u
Sun Aug 21 14:00:04 UTC 2016
```

After 9, you'll see `node2` become standby.

```
Last updated: Sun Aug 21 14:00:13 2016          Last change: Sun Aug 21 14:00:10 2016 by root via crm_attribute on node1
Stack: corosync
Current DC: node2 (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
2 nodes and 3 resources configured

Online: [ node1 node2 ]

 Master/Slave Set: pgsql-master [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
master-ip       (ocf::heartbeat:IPaddr2):       Started node1

Node Attributes:
* Node node1:
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 0000000003000398
    + pgsql-status                      : PRI
* Node node2:
    + master-pgsql                      : 100
    + pgsql-data-status                 : STREAMING|SYNC
    + pgsql-status                      : HS:sync

Migration Summary:
* Node node2:
* Node node1:

Failed Actions:
* pgsql_start_0 on node2 'unknown error' (1): call=23, status=complete, exitreason='My data may be inconsistent. You have to remove /va
r/run/postgresql/PGSQL.lock file to force start.',
    last-rc-change='Sun Aug 21 13:58:23 2016', queued=0ms, exec=383ms
```



## License
MIT
