# What is this?

Single-sentence description? Start a kubernetes cluster with a dedicated certificate-auth enabled etcd cluster, IPsec between nodes, and glusterfs on Linode by running a python script and a few ansible playbooks. That is probably more interesting to people than what I'm running on the end result.

The ansible directory contains the `up.py` script and playbooks used to bootstrap the environment.

This manifests directory contains configuration to run several applications using kubernetes, and to access them from the web.
The applications are:
1. Single-serving site "whatcolorischristinashair.com", the original purpose for this whole idea
2. [Mosquitto](https://mosquitto.org) MQTT broker
3. [owntracks2db](https://github.com/vigevenoj/owntracks-to-db), for storing [Owntracks](owntracks.org) location updates into a database


# Prerequisites

Really, all you need are
1. A Linode API token
2. A computer with python 3 and ansible
3. A private certificate authority, or the ability to generate some certificates
since the preflight will give you virtual machines to run it on

# Preflight

This is applicable to people who want to run a Kubernetes cluster on a group of virtual machines at Linode.

## Cluster creation
You'll need a Linode account and a valid API token in addition to some other details in a configuration file that you should not check into source control. An example config.yaml:
```
linode:
  token: 
  # Use this to group all of the VMs used for your cluster
  group: fancy-birdbath
  region: us-east-1a
  # g5-standard-1 is 2048mb ram/1vcpu, $10/month
  # g5-nanode-1 is 1024mb ram/1vcpu, $5/month
  type: g5-standard-1
  distro: linode/ubuntu17.04
  root_pass: 
  root_ssh_key: 

# Put your kubernetes cluster information here, we'll use it later
cluster:
  # This is the number of nodes in the cluster, total (including master)
  nodes: 3
  # The cluster requires a domain that has DNS configured via Linode
  domain: example.com
```
With your config in place, run `python up.py` to generate your cluster nodes and ansible inventory as specified.

Next, you need use a certificate authority to generate certificates (and keys...) for each member of the cluster. Add those to ansible/roles/ipsec/files, along with the certificate of the certificate authority that signed them. I'm glossing over this step because I run a CA for personal projects, but you can use something like https://github.com/radiac/caman to do this step.

# Cluster creation

Now run the playbooks:
1. playbooks/minimal-bootstrap.yml: This must be run once, prior to any other playbooks, in order to ensure that every host has a unique machine id
1. site.yml: This configures a baseline for each host, configures certificate-base IPsec transport between cluster members, builds an etcd cluster with certificate authentication on three hosts, and configures kubelet on the hosts which will be kubernetes nodes.
1. kubeadm.yml: This bootstraps the kubernetes cluster using kubeam. Following this playbook, `kubectl get nodes` should return all hosts as nodes in Ready state
1. storage.yml: This playbook labels the kubernetes nodes as storage nodes, creates a daemonset for glusterfs, and creates a Heketi cluster for volume management using the topology generated by the up.py script run initially.


Note that the playbooks require the controller (local machine) to have the python 'netaddr' package installed.

### Inter-node traffic encryption
The ansible playbook configures certificate-based IPsec encapsulation of traffic between the nodes on their internal (private) addresses via the `ipsec` role. Certificates need to be generated ahead of time, and the chain of trust up to a root needs to be copied onto the nodes in addition to each node's certificate and key. There are some notes in the role with more details.

### Storage
Persistent storage is managed via 20gb volumes attached to nodes as unformatted block devices. This is handled via some internal tooling during the preflight. It generates the necessary topology.json required for Heketi to use the volumes for glusterfs.  
Once the cluster is boostrapped and Kubernetes is running, the `storage` playbook runs the gk-deploy script from [gluster-kubernetes](https://github.com/gluster/gluster-kubernetes) and heketi-cli from [Heketi](https://github.com/heketi/heketi). 

Once storage is online, create a StorageClass to fulfill prerequisite #3. [storageclass.yaml](storageclass.yaml) will work for this, but requires the IP address of the Heketi service from kubernetes


### Status Check
* minimal-bootstrap.yml playbook: after running this, the /etc/machine-id should be different from the other nodes
* site.yml
  * All nodes will have certificate-based IPsec transport between them
  * All nodes have etcd installed and configured to use the same certificates as IPsec traffic
  * The etcd cluster has been established
  * All nodes are ready for kubeadm
* kubeadm.yml
  * First node is configured as kubernetes master
  * Remaining nodes are configured as kubernetes workers
  * Kubernetes cluster is up and running
* storage.yml: heketi bootstraps glusterfs cluster on block storage devices

# Service and Application Deployment
## Deploy postgresql with helm:

   `helm install --name basic-database stable/postgresql`

## Deploy Traefik

   First, update the ConfigMap in traefik-configmap.yaml to include the domains you plan on managing via Traefik, as SSL is obtained via Let's Encrypt.  
   Next, ensure that a persistent volume claim exists for Traefik to store its certificate information by applying the traefik-pvc.yaml:  
   `kubectl apply -f traefik-pvc.yaml`  
   Then apply the ConfigMap you updated previously  
   `kubectl apply -f traefik-configmap.yaml`  
   ensure that Traefik has the necessary roles assigned,  
   `kubectl apply -f traefik-rbac.yaml`  

   Create a deployment for the controller:
   `kubectl apply -f traefik-ingress-controller_deployment.yaml`
## Deploy Traefik ui service

   `kubectl apply -f traefik-ui_service.yaml`  
 
## Deploy whatcolorischristinashair

   This is both a joke and the reason that this project exists. For additional details, see [whatcolorischristinashair](https://github.com/vigevenoj/whatcolorischristinashair). The deployment and service are managed via haircolor.yaml and a Traefik ingress is managed via haircolor-ingress.yaml  

   `kubectl apply -f haircolor.yaml`  
   `kubectl apply -f haircolor-ingress.yaml`  

### Updating side-loaded images that are not publicly available
   For images that are built privately and then side-loaded into the cluster, the following steps need to happen:  
   1. Build the docker image with `docker build`
   2. Save the image locally with `docker save`
   3. scp the image to the cluster nodes
   4. Load the image into the cluster nodes' local registry with `docker load`



# Future work
in no particular order
* Add inbound/outbound SMTP
* mqtt (see mosquitto.yaml for this. It does not have an ingress)
* owntracks (owntracks2db pod works and is persisting data, see locationupdates.yaml)


# References
* Overview/deployment
  * https://5pi.de/2016/11/20/15-producation-grade-kubernetes-cluster/
  * http://jeremievallee.com/2017/01/31/kubernetes-with-vagrant-ansible-kubeadm/
  * http://blog.jameskyle.org/2014/08/deploying-baremetal-kubernetes-cluster/
  * https://kubernetes.io/docs/setup/independent/high-availability
  * http://michele.sciabarra.com/2018/02/12/devops/Kubernetes-with-KubeAdm-Ansible-Vagrant/
* Storage (specifically GlusterFS/Heketi)
  * https://github.com/gluster/gluster-kubernetes/tree/master/vagrant
  * https://github.com/heketi/heketi/wiki/Kubernetes-Integration
  * http://dougbtv.com/nfvpe/2017/04/05/glusterfs-persistent/
  * http://blog.lwolf.org/post/how-i-deployed-glusterfs-cluster-to-kubernetes/
  * http://blog.lwolf.org/post/how-to-recover-data-from-broken-glusterfs-cluster/
* Ingress
  * http://containerops.org/2017/01/30/kubernetes-services-and-ingress-under-x-ray/
* Other options
  * https://blog.hypriot.com/post/setup-kubernetes-raspberry-pi-cluster/ for at home
* Linode Python API: https://github.com/linode/python-linode-api/tree/master/linode
* IPsec configuration
  * https://www.gypthecat.com/easyish-ipsec-vpn-with-shared-ecdsa-certificates-for-host-to-host-connections
  * https://github.com/kravietz/ansible-ipsec
  * https://github.com/rnurgaliyev/ansible-vpn-ikev2
  * https://github.com/networklore/ansible-getting-started
  * https://github.com/torian/ansible-role-strongswan
  * https://ipsec.pl/ipsec/2016/securing-cloud-servers-ipsec-and-ansible.html
  * https://github.com/jonathanio/ansible-role-strongswan
  * strongSwan documentation at https://strongswan.org/
* etcd
  * https://stackoverflow.com/questions/44631245/how-to-run-etcd-cluster-in-docker-using-ansible#44632366

## Notes

### Adding basic auth for an ingress
 * Create credentials with `htpasswd -Bc ingressname.auth.secret username`
 * `kubectl create secret generic ingressname-basic-auth --from-file ingressname.auth.secret`

# Troubleshooting

## Copying data from a gluster-backed volume
 * Translate PVC and PV to the Heketi volume name by cutting off the first four characters of the output of `kubectl get pv $(kubectl get pvc pvcname -o jsonpath='{.spec.volumeName}{"\n"}') -o jsonpath='{.spec.glusterfs.path}{"\n"}'`
 * Use the `heketi-cli` API or `gluster` command (via `kubectl exec` on a glusterfs pod) to determine name of the brick where the data resides and which host that brick is on.
 * On the host where the brick is located, use `lvdisplay` to find the path of the LV where the brick is
 * Mount the device from the LV Path of `lvdisplay` somewhere, eg, /mount/mybrick
 * Copy whatever data you need from /mount/mybrick/brick/

## All gluster volumes have gone read-only
This is usually caused by an unplanned reboot of the physical hardware under the underlying host VM.  Note that the gluster logs may only indicate some sort of networking problem (connection refused to the remote bricks making up the rest of the volume) but the problem is that the process that should be listening is not running on the remote hosts.
 * Confirm that there are fewer than two processes involving gluster on the VM with `ps aux | grep gluster`
 * Delete the existing glusterfs pods and let them be re-created to resolve the issue

## Certificates have expired
Don't let this happen. See https://github.com/kubernetes/kubeadm/issues/581#issuecomment-421477139 for a partial resolution. Rebooting the nodes after deploying new certificates to a 1.10 cluster may result in the nodes losing their keys, requiring either re-generation of keys or rebuilding the cluster. Check for ```Jun 26 05:32:45 robot-ghost-poop-2 kubelet[24333]: E0626 05:32:45.781780   24333 bootstrap.go:179] Unable to read existing bootstrap client config: invalid configuration: [unable to read client-cert /var/lib/kubelet/pki/kubelet-client.crt for default-auth due to open /var/lib/kubelet/pki/kubelet-client.crt: no such file or directory, unable to read client-key /var/lib/kubelet/pki/kubelet-client.key for default-auth due to open /var/lib/kubelet/pki/kubelet-client.key: no such file or directory]
Jun 26 05:32:45 robot-ghost-poop-2 kubelet[24333]: F0626 05:32:45.821314   24333 server.go:233] failed to run Kubelet: cannot create certificate signing request: Unauthorized``` for additional troubleshooting.

# Relevant bug reports
* https://github.com/kubernetes/kubernetes/issues/41141 : "Transport endpoint is not connected" results in pods with glusterfs volumes stuck in Terminating state until the node is rebooted or the mount is manually unmounted via `fusermount -uz [path to mount]`
* Heketi needs an exclusive lock on its database (boltdb). If an existing pod terminates and a new pod replaces it quickly enough, the second pod will be unable to access the database in read/write mode and will log a message that it is trying again in read-only mode. If this happens, existing volume claims will be honored but new ones cannot be created. To resolve it, scale the Heketi deployment to 0 replicas with `kubectl scale --replicas=0 deploy/heketi`, wait a few minutes, and then scale the deployment back to 1 replica with `kubectl scale --replicas=1 deploy/heketi` and then watch its logs to confirm Heketi started up in read/write mode. See https://github.com/gluster/gluster-kubernetes/issues/257 for some additional details
