# Overview
This set of playbooks upgrades MySQL from 5.5 to 5.6 in a running cluster.
It does this live while leaving the cluster live. Do not attempt to upgrade
a dead node (per the mysql directions).

The theory here is that one node is left up and running mysql 5.5, and during
this time the other nodes are all upgraded. After being upgraded to 5.6, the nodes
are left in a read_only mode. Then lastly we upgraded node 1. This final node upgrade
will cause an outage since nodes 2 & 3 are still readonly. After Node 1 is done,
we dynamically turn read-only off on nodes 2 & 3 and we're done.

http://www.percona.com/doc/percona-xtradb-cluster/5.6/upgrading_guide_55_56.html

# Usage

This script requires that you pass in the blob version that you want to use for the
latest Percona repo as follows.

    ansible-playbook -e=blob_version VERSION oneoff/upgrade_mysql/upgrade-mysql-on-control-nodes.yaml

# Playbooks & Ordering

0) Disable puppet and let the runs finish.

1) playbooks/control-prep.yaml

* Run cluster check to ensure the cluster is healthy.
* move the Percona repo pointer to a newer date so we get the latest copy of MySQL+Percona s/w
* Run apt-get update

2) playbooks/control-prep_other_nodes.yaml

This script preps the upgrade. It does this via the following steps:

* ensure galera is healthy on all nodes
* stop mysql on all but node 1
* backup mysql on all but node 1
* ensure galera is healthy on node 1

3) playbooks/control-upgrade_other_nodes.yaml

This script upgrades mysql on nodes 2 & 3.

* remove old mysql packages
* set read_only=ON in an upgrade config file, puppet will later purge this.
* install the new mysql packages

4) playbooks/control-_prep_node1.yaml

This script preps the upgrade on Node 1. It does this via the following steps:

* shutdown mysql

5) playbooks/control-upgrade_node1.yaml

This script upgrades mysql on the remaining node. There will be a mysql outage
during this run.

* Stop mysql on node 1. The cluster is now down.
* Remove old packages on node 1.
* Install new packages on node 1.
* The cluster is now up.
* run mysql_upgrade on node 1

6) playbooks/control-disable_readonly.yaml

Finally we need to turn read_only off on nodes 2 & 3

* Dynamically disable read_only on nodes 2 & 3
* Remove the config file that set readonly so that if mysql restarts it wont come up readonly.
