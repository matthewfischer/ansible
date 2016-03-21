# Overview
This set of playbooks is to support Keystone upgrades that require database
migrations that are not backwards compatible.  For these sort of upgrades,
database migrations must be done while no keystone services are running. 
These will require a brief outage while puppet upgrades your node and runs
the database migrations.

# Playbooks

## begin-keystone-upgrade.yaml

The begin-keystone-upgrade.yaml playbook does the following:

 * Disable Puppet on all sites and wait for any existing catalog runs to complete

 * Check out the given deploy ref on build servers in all sites

 * Disable Keystone in site2

 * Disable Galera/MySQL on the first keystone node in site1

 * Disable Keystone in site1 except on the first keystone node

 * Run Puppet on the first keystone node in site1

It's intended that this playbook will be used to begin the the upgrade process.
Once this playbook is run, Keystone will be upgraded on the first keystone node
in 'site1'.  Once this is complete, validation of Keystone functionality should
be completed, then a normal deployment should proceed to upgrade the remaining
nodes.

## start-galera-and-keystone.yaml

This playbook will start and Keystone MySQL/Galera services on keystone zones.
This can be useful during testing to start services after a failed run or test.

# Starting the upgrade

These playbooks are intended to be used via the `ansible-playbook_adhoc`
Jenkins job. To run the upgrade job you should pass the following parameters
into that job:

    -l <echelon> oneoff/keystone-upgrade/begin-keystone-upgrade.yaml -e deploy_ref=<TAG>

If you are running this by hand, you need to specify the multi-site inventory:

    ansible-playbook -l <echelon> -i support/multi-site-inventory.py -e deploy_ref=<TAG>

The `<echelon>` should be replaced with dev, staging or prod.  The `<TAG>`
should be replaced with the OPSDEPLOY tag for the deployment.

After this, validate the first node in site1 is functioning properly. This is your
chance to abort the upgrade. Once this node is validated and you move on to
the others, it gets more difficult to undo the damage.

Complete the Keystone upgrade with a normal keystone node only deploy, using
-l keystone in Jenkins. You need to do both regions. You can do them in any
order but doing site2 (West) first might make more sense so that you have
Keystone up in both sites.

After doing the keystone-node only deploy, you need to re-enable puppet on all
nodes. This can be done by running the `ansible-playbook_adhoc` Jenkins job
with these parameters:

    -l <echelon> playbooks/puppet-start.yaml

# Aborting and Cleanup

If the upgrade of the first node fails or the new package is determined to be
broken, here is the process for undoing the damage. YOUR GOAL IS TO GET TO
STEP 4 AS SOON AS POSSIBLE TO END THE OUTAGE. In parallel to these
steps someone should be working on a revert of the upgrade change.

Example nodes which will be used during this discussion:

Site1:
node1: east01-keystone-001
node2: east01-keystone-002
node3: east01-keystone-003

Site2:
node1: west01-keystone-001
node2: west01-keystone-002
node3: west01-keystone-003

Under this scenario, after this job runs you should be in the following state:

Keystone running with upgraded copy: site1-node1
Mysql running: All of site1
Keystone & MySQL stopped: All of Site2
Puppet running: site1-node1
Puppet not running: all but site1-node1

1) Stop puppet, mysql, and keystone on site1-node-1. Assuming everything was at
least semi-functional before, you are now in an OUTAGE.

2) Stop mysql on site1-node2 and site2-node3. Ensure the entire keystone cluster
is down.

3) Bootstrap mysql on site2-node1.

4) Start keystone on site2-node1. Outage is over. Test that keystone is working.

5) Start mysql on site2-node2, wait for clustercheck. Repeat on site2-node3.

6) Start keystone on site2-node2, site2-node3. Test keystone again.

7) rm -rf /var/lib/mysql/\* on site1-node2 and site1-node3. These nodes have
bad "upgraded" data.

8) Start mysql on site1-node2, wait for clustercheck. Repeat on site1-node3.
This will be a full database copy and may take several minutes per node.

9) Start keystone on site1-node2 and site1-node3.

10) dd the box on site1-node1. Power the box off.

11) Land and deploy the revert.

12) Re-PXE and re-install site1-node1.
