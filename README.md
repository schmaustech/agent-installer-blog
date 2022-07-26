# **Meet The New Agent-Based OpenShift Installer**

<img src="agent-based.png" style="width: 1000px;" border=0/>

With OpenShift 4.11 we are introducing a pre-release version of the new agent-based installer for OpenShift. Installing clusters on bare metal nodes has never been so easy. Our aim is to provide the flexibility of user-provided infrastructure (UPI) installs with the ease of use of the OpenShift Assisted Installer, while in fully disconnected or air-gapped environments. We have worked with users from different industries and incorporated their feedback and use cases in its design. To do this we are combining existing technologies and experience coming from the Assisted Installer and the installer-provisioned infrastructure (IPI). 

## Some of the use cases we are addressing

* Quickly deploy isolated OpenShift clusters on premise of any topology
* Install a cluster zero with OpenShift management components such as Red Hat Advanced Cluster Management for Kubernetes (ACM), GitOps, Quay, and components to support your other clusters.
* As we start with Hypershift, install the OpenShift cluster that will house the control planes for all the other clusters.

## Some of the features we are including and working on

* Create bootable images with openshift-install to deploy your clusters
* In-place bootstrap, no extra node required
* Works in fully disconnected deployments
* Works with a mirrored local registry
* Supports single node OpenShift (SNO), compact 3-node clusters, and highly available topologies
* Can be automated with third party tools
* User-friendly interface based on the Assisted Installer

## Getting Familiar with Agent-bBased OpenShift Installer

Workflow

<img src="node-lifecycle.png" style="width: 1000px;" border=0/>

How to get binary

~~~bash
$ cd ~/
$ mkdir ~/manifests


With the directory created we can move onto creating the agent cluster install resource file.  This file specifies the clusters configuration such as number of control plane and/or worker nodes, the api and ingress vip and the cluster networking.   In my example I will be deploying a 3 node compact cluster which referenced a cluster deployment named kni22:

~~~bash
$ cat << EOF > ./manifests/agent-cluster-install.yaml
apiVersion: extensions.hive.openshift.io/v1beta1
kind: AgentClusterInstall
metadata:
  name: kni22
  namespace: kni22
spec:
  apiVIP: 192.168.0.125
  ingressVIP: 192.168.0.126
  clusterDeploymentRef:
    name: kni22
  imageSetRef:
    name: openshift-v4.10.0
  networking:
    clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
    serviceNetwork:
    - 172.30.0.0/16
  provisionRequirements:
    controlPlaneAgents: 3
    workerAgents: 0 
  sshPublicKey: 'INSERT PUBLIC SSH KEY HERE'
EOF
~~~

Next we will create the cluster deployment resource file which defines the cluster name, domain, and other details:

~~~bash
$ cat << EOF > ./manifests/cluster-deployment.yaml
apiVersion: hive.openshift.io/v1
kind: ClusterDeployment
metadata:
  name: kni22
  namespace: kni22
spec:
  baseDomain: schmaustech.com
  clusterInstallRef:
    group: extensions.hive.openshift.io
    kind: AgentClusterInstall
    name: kni22-agent-cluster-install
    version: v1beta1
  clusterName: kni22
  controlPlaneConfig:
    servingCertificates: {}
  platform:
    agentBareMetal:
      agentSelector:
        matchLabels:
          bla: aaa
  pullSecretRef:
    name: pull-secret
EOF
~~~

Moving on we now create the cluster image set resource file which contains OpenShift image information such as the repository and image name.  This will be the version of the cluster that gets deployed in our 3 node compact cluster.  In this example we are using 4.10.23:

~~~bash
$ cat << EOF > ./manifests/cluster-image-set.yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: ocp-release-4.10.23-x86-64-for-4.10.0-0-to-4.11.0-0
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.10.23-x86_64
EOF
~~~

Next we define the infrastructure environment file which  contains information for pulling OpenShift onto the target host nodes we are deploying to:

~~~bash
$ cat << EOF > ./manifests/infraenv.yaml 
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: kni22
  namespace: kni22
spec:
  clusterRef:
    name: kni22  
    namespace: kni22
  pullSecretRef:
    name: pull-secret
  sshAuthorizedKey: 'INSERT PUBLIC SSH KEY HERE'
  nmStateConfigLabelSelector:
    matchLabels:
      kni22-nmstate-label-name: kni22-nmstate-label-value
EOF
~~~

Generate install-config.yaml

~~~bash
Install-config.yaml
~~~

Create image

Move image to nodes

Once the agent create image command completes we are left with a agent.iso image which is in fact our OpenShift install ISO:

~~~bash
$ ls -l ./output/
total 1073152
-rw-rw-r--. 1 bschmaus bschmaus 1098907648 May 20 08:55 agent.iso
~~~

Since the nodes I will be using to demonstrate this 3 node compact cluster are virtual machines all on the same KVM hypervisor I will go ahead and copy the agent.iso image over to that host:

~~~bash
$ scp ./output/agent.iso root@192.168.0.22:/var/lib/libvirt/images/
root@192.168.0.22's password: 
agent.iso 
~~~

With the image moved over to the hypervisor host I went ahead and ensured each virtual machine we are using (asus3-vm[1-3]) has the image set.  Further the hosts are designed boot off the ISO if the disk is empty.  We can confirm everything is ready with the following output:

~~~bash
# virsh list --all
 Id   Name        State
----------------------------
 -    asus3-vm1   shut off
 -    asus3-vm2   shut off
 -    asus3-vm3   shut off
 -    asus3-vm4   shut off
 -    asus3-vm5   shut off
 -    asus3-vm6   shut off

# virsh domblklist asus3-vm1
 Target   Source
---------------------------------------------------
 sda      /var/lib/libvirt/images/asus3-vm1.qcow2
 sdb      /var/lib/libvirt/images/agent.iso

# virsh domblklist asus3-vm2
 Target   Source
---------------------------------------------------
 sda      /var/lib/libvirt/images/asus3-vm2.qcow2
 sdb      /var/lib/libvirt/images/agent.iso

# virsh domblklist asus3-vm3
 Target   Source
---------------------------------------------------
 sda      /var/lib/libvirt/images/asus3-vm3.qcow2
 sdb      /var/lib/libvirt/images/agent.iso
~~~

Now lets go ahead and start the first virtual machine:

~~~bash
# virsh start asus3-vm1
Domain asus3-vm1 started
~~~

Once the first virtual machine is started we can switch over to the console and watch it boot up:

<img src="asus3-vm1-console.png" style="width: 800px;" border=0/>

Deploy
