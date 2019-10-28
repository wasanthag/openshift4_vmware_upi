# Deploy Openshift 4.x on VMWare user provisioned Infrastructure
This playbooks are tested with OpenShift 4.1 and 4.2 with Vsphere 6.5.0

## Prerequisites
- HAProxy (or any other loadbalancer) is setup to loadbalance the OpenShift API endpoints

*Note that in haproxy loadbalancer, it is required that API VIPs to point to the bootstrap server initially. These lines are commented out
in the configs. When you first start to run the playbooks provision-upi.yaml , they should be uncommented.*

A sample haproxy config is included in the repo.

- DNS entries required by the OpenShift

A nice example of DNS entries required are listed here, https://blog.openshift.com/openshift-4-bare-metal-install-quickstart/

- rhcos vmware ova is uploaded in the vm folder

- openshift-install, oc client and kubectl binaries are downloaded on the server running ansible and in the path.

## Update the parameters
- Update the parameters in group_vars/all/vars.yaml that describes the Vsphere environment as well as OpenShift deployment parameters.

*pull secret is only valid for 24 hours and can be pulled from https://cloud.redhat.com/openshift/install*

*ignition_local_dir is where the generated ignition files will be stored.*

*web_host is where the ignition file for the bootstrap host will be stored.*

## Create Ignition files
```
ansible-playbook generate-ignition.yaml -v
```
This will create a local directory with ignition files and upload bootstrap.ign on to a webserver
This will also create append-bootstrap.save and install-config.save files generated for review inside the ansible directory.These are useful for review for any
typos etc.. as the next playbook provision-upi will delete the generated yaml files inside the ignition directory.

## Create VMWare Nodes
```
ansible-playbook provision-upi.yaml -v
```
This will create bootstrap, masters and computes and leave them power off
## Update DNS/DHCP
Update the IPAM to reflect the DHCP fixed address records with MAC addresses noted from the VMs.

## Power on the Nodes
```
ansible-playbook upi-power-on.yaml -v
```
Once the nodes are powered on, bootstrap node will boot and start kubeconfig process. All other nodes will boot and bootstrap
from the bootstrap node.
At this point you can log in to the bootstrap (ssh core@<bootstrap node>) and watch the cluster configuration being done with 
```
journalctl -b -f -u bootkube.service
```
If sucessfully boot strapped, following should be displyed in the logs
```
Oct 28 15:52:58 bootstrap.ocp4c1.cloud.mycompany.local bootkube.sh[1748]: Tearing down temporary bootstrap control plane...
Oct 28 15:52:58 bootstrap.ocp4c1.cloud.mycompany.local bootkube.sh[1748]: bootkube.service complete

```
Also you can log in to masters and watch the logs for progress of forming the etcd cluster 
```
journalctl -f
```

By this time the cluster is most likely boot strapped. You can confirm this by 
```
cd <ignition_local_dir/auth>
export KUBECONFIG=./kubeconfig
oc get nodes
oc get co

```

## Run the Openshift cluster create
```
ansible-playbook bootstrap-cluster.yaml -v
```
This will run the openshift install to create cluster. Most likely at this point the cluster is bootstrapped and this will finish quickly.

## Remove the loadbalancer entries for the bootstrap node from the loadbalancer
Edit haproxy/loadbalancer configs and remove bootstrap node from api and api-int pools.

## Check the OCP cluster health
Confirm all cluster operators are ready.
```
ansible-playbook check-cluster-status.yaml -v
```
## Finalize the install
```
ansible-playbook finalize-install.yaml -v
```

## Misc
Use the following playbook to delete the UPI nodes from Vsphere

```
ansible-playbook delete-upi-nodes.yaml -v
```
