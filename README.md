# **Meet The New Agent-Based OpenShift Installer**

<img src="agent-based.png" style="width: 1000px;" border=0/>

With OpenShift 4.11 we are introducing a pre-release version of the new agent-based installer for OpenShift. Installing clusters on bare metal nodes has never been so easy. Our aim is to provide the flexibility of user-provided infrastructure (UPI) installs with the ease of use of the OpenShift Assisted Installer, while in fully disconnected or air-gapped environments. We have worked with users from different industries and incorporated their feedback and use cases in its design. To do this we are combining existing technologies and experience coming from the Assisted Installer and the installer-provisioned infrastructure (IPI). 

## Use Cases We are Addressing

* Quickly deploy isolated OpenShift clusters on premise of any topology
* Install a cluster zero with OpenShift management components such as Red Hat Advanced Cluster Management for Kubernetes (ACM), GitOps, Quay, and components to support your other clusters.
* As we start with Hypershift, install the OpenShift cluster that will house the control planes for all the other clusters.

## Features Included and/or In Progress

* Create bootable images with openshift-install to deploy your clusters
* In-place bootstrap, no extra node required
* Works in fully disconnected deployments
* Works with a mirrored local registry
* Supports single node OpenShift (SNO), compact 3-node clusters, and highly available topologies
* Can be automated with third party tools
* User-friendly interface based on the Assisted Installer

## Getting Familiar with Agent-Based OpenShift Installer

Write something about the workflow describing the image below

<img src="node-lifecycle.png" style="width: 1000px;" border=0/>

Now that we understand how the Agent-Based Installer works and what use cases its trying to solve lets focus on actually using it.   As of this writing the Agent-based Installer needs to be compiled from the forked installer source code.  This requirements will go away once OpenShift 4.12 goes GA at which time the Agent-based Installer will be GA as well.

So lets go ahead and grab the OpenShift installer source code from Github and checkout the agent-installer branch:

~~~bash
$ git clone https://github.com/openshift/installer
Cloning into 'installer'...
remote: Enumerating objects: 204497, done.
remote: Counting objects: 100% (210/210), done.
remote: Compressing objects: 100% (130/130), done.
remote: Total 204497 (delta 99), reused 153 (delta 70), pack-reused 204287
Receiving objects: 100% (204497/204497), 873.44 MiB | 10.53 MiB/s, done.
Resolving deltas: 100% (132947/132947), done.
Updating files: 100% (86883/86883), done.

$ cd installer/
$ git checkout agent-installer
Branch 'agent-installer' set up to track remote branch 'agent-installer' from 'origin'.
Switched to a new branch 'agent-installer'

$ git branch
* agent-installer
  master
~~~

Once we have the source code checked out we need to go ahead and build the the OpenShift install binary:

~~~bash
$ hack/build.sh
+ minimum_go_version=1.17
++ go version
++ cut -d ' ' -f 3
+ current_go_version=go1.17.7
++ version 1.17.7
++ IFS=.
++ printf '%03d%03d%03d\n' 1 17 7
++ unset IFS
++ version 1.17
++ IFS=.
++ printf '%03d%03d%03d\n' 1 17
++ unset IFS
+ '[' 001017007 -lt 001017000 ']'
+ make -C terraform all
make: Entering directory '/home/bschmaus/installer/terraform'
cd providers/alicloud; \
(...)
make: Leaving directory '/home/bschmaus/installer/terraform'
+ copy_terraform_to_mirror
++ go env GOOS
++ go env GOARCH
+ TARGET_OS_ARCH=linux_amd64
+ rm -rf '/home/bschmaus/installer/pkg/terraform/providers/mirror/*/'
+ find /home/bschmaus/installer/terraform/bin/ -maxdepth 1 -name 'terraform-provider-*.zip' -exec bash -c '
      providerName="$(basename "$1" | cut -d - -f 3 | cut -d . -f 1)"
      targetOSArch="$2"
      dstDir="${PWD}/pkg/terraform/providers/mirror/openshift/local/$providerName"
      mkdir -p "$dstDir"
      echo "Copying $providerName provider to mirror"
      cp "$1" "$dstDir/terraform-provider-${providerName}_1.0.0_${targetOSArch}.zip"
    ' shell '{}' linux_amd64 ';'
Copying alicloud provider to mirror
Copying aws provider to mirror
Copying azureprivatedns provider to mirror
Copying azurerm provider to mirror
Copying azurestack provider to mirror
Copying google provider to mirror
Copying ibm provider to mirror
Copying ignition provider to mirror
Copying ironic provider to mirror
Copying libvirt provider to mirror
Copying local provider to mirror
Copying nutanix provider to mirror
Copying openstack provider to mirror
Copying ovirt provider to mirror
Copying random provider to mirror
Copying vsphere provider to mirror
Copying vsphereprivate provider to mirror
+ mkdir -p /home/bschmaus/installer/pkg/terraform/providers/mirror/terraform/
+ cp /home/bschmaus/installer/terraform/bin/terraform /home/bschmaus/installer/pkg/terraform/providers/mirror/terraform/
+ MODE=release
++ git rev-parse --verify 'HEAD^{commit}'
+ GIT_COMMIT=d74e210f30edf110764d87c8223a18b8a9952253
++ git describe --always --abbrev=40 --dirty
+ GIT_TAG=unreleased-master-6040-gd74e210f30edf110764d87c8223a18b8a9952253
+ DEFAULT_ARCH=amd64
+ GOFLAGS=-mod=vendor
+ LDFLAGS=' -X github.com/openshift/installer/pkg/version.Raw=unreleased-master-6040-gd74e210f30edf110764d87c8223a18b8a9952253 -X github.com/openshift/installer/pkg/version.Commit=d74e210f30edf110764d87c8223a18b8a9952253 -X github.com/openshift/installer/pkg/version.defaultArch=amd64'
+ TAGS=
+ OUTPUT=bin/openshift-install
+ export CGO_ENABLED=0
+ CGO_ENABLED=0
+ case "${MODE}" in
+ LDFLAGS=' -X github.com/openshift/installer/pkg/version.Raw=unreleased-master-6040-gd74e210f30edf110764d87c8223a18b8a9952253 -X github.com/openshift/installer/pkg/version.Commit=d74e210f30edf110764d87c8223a18b8a9952253 -X github.com/openshift/installer/pkg/version.defaultArch=amd64 -s -w'
+ TAGS=' release'
+ test '' '!=' y
+ go generate ./data
writing assets_vfsdata.go
+ echo ' release'
+ grep -q libvirt
+ go build -mod=vendor -ldflags ' -X github.com/openshift/installer/pkg/version.Raw=unreleased-master-6040-gd74e210f30edf110764d87c8223a18b8a9952253 -X github.com/openshift/installer/pkg/version.Commit=d74e210f30edf110764d87c8223a18b8a9952253 -X github.com/openshift/installer/pkg/version.defaultArch=amd64 -s -w' -tags ' release' -o bin/openshift-install ./cmd/openshift-install
~~~

Now that we have the binary lets go ahead and create a directory that will contain the required manifests we need for our deployment that will get injected into the ISO we build:

~~~bash
$ cd ~/
$ mkdir ~/manifests
~~~

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

The next file is the nmstate configuration file and this file provides all the details for all of the host that will be booted using the ISO image we are going to create.   Since we have a 3 node compact cluster to deploy we notice that in the file below we have specified three nmstate configurations.  Each configuration is for a node and generates a static IP address on the nodes enp2s0 interface that matches the MAC address defined.   This enables the ISO to boot up and not necessarily require DHCP in the environment which is what a lot of customers are looking for.   Again my example has 3 configurations but if we had worker nodes we would add those in too.   Lets go ahead and create the file:

~~~bash
$ cat << EOF > ./manifests/nmstateconfig.yaml
---
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: mynmstateconfig01
  namespace: openshift-machine-api
  labels:
    kni22-nmstate-label-name: kni22-nmstate-label-value
spec:
  config:
    interfaces:
      - name: enp2s0
        type: ethernet
        state: up
        mac-address: 52:54:00:e7:05:72
        ipv4:
          enabled: true
          address:
            - ip: 192.168.0.116
              prefix-length: 24
          dhcp: false
    dns-resolver:
      config:
        server:
          - 192.168.0.10
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 192.168.0.1
          next-hop-interface: enp2s0
          table-id: 254
  interfaces:
    - name: "enp2s0"
      macAddress: 52:54:00:e7:05:72
---
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: mynmstateconfig02
  namespace: openshift-machine-api
  labels:
    kni22-nmstate-label-name: kni22-nmstate-label-value
spec:
  config:
    interfaces:
      - name: enp2s0
        type: ethernet
        state: up
        mac-address: 52:54:00:95:fd:f3
        ipv4:
          enabled: true
          address:
            - ip: 192.168.0.117
              prefix-length: 24
          dhcp: false
    dns-resolver:
      config:
        server:
          - 192.168.0.10
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 192.168.0.1
          next-hop-interface: enp2s0
          table-id: 254
  interfaces:
    - name: "enp2s0"
      macAddress: 52:54:00:95:fd:f3
---
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: mynmstateconfig03
  namespace: openshift-machine-api
  labels:
    kni22-nmstate-label-name: kni22-nmstate-label-value
spec:
  config:
    interfaces:
      - name: enp2s0
        type: ethernet
        state: up
        mac-address: 52:54:00:e8:b9:18
        ipv4:
          enabled: true
          address:
            - ip: 192.168.0.118
              prefix-length: 24
          dhcp: false
    dns-resolver:
      config:
        server:
          - 192.168.0.10
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 192.168.0.1
          next-hop-interface: enp2s0
          table-id: 254
  interfaces:
    - name: "enp2s0"
      macAddress: 52:54:00:e8:b9:18
EOF
~~~

The final file we need to create is the pull-secret resource file which contains the pull-secret values so that our cluster can pull in the required OpenShift images to instantiate the cluster:

~~~bash
$ cat << EOF > ./manifests/pull-secret.yaml 
apiVersion: v1
kind: Secret
type: kubernetes.io/dockerconfigjson
metadata:
  name: pull-secret
  namespace: kni22
stringData:
  .dockerconfigjson: 'INSERT JSON FORMATTED PULL-SECRET'
EOF
~~~

At this point we should now have our six required files defined to build our Agent Installer ISO:

~~~bash
$ ls -1 ./manifests/
agent-cluster-install.yaml
cluster-deployment.yaml
cluster-image-set.yaml
infraenv.yaml
nmstateconfig.yaml
pull-secret.yaml 
~~~

Generate install-config.yaml

~~~bash
Install-config.yaml
~~~

We are now ready to use the Openshift install binary we compiled earlier with the Agent Installer code to generate our ephemeral OpenShift ISO.   We do this by issuing the following command which introduces the agent option.  This in turn will read in the manifest details we generated and download the corresponding RHCOS image and then inject our details into the image writing out a file called agent.iso:

~~~bash
$ bin/openshift-install agent create image 
INFO adding MAC interface map to host static network config - Name:  enp2s0  MacAddress: 52:54:00:e7:05:72 
INFO adding MAC interface map to host static network config - Name:  enp2s0  MacAddress: 52:54:00:95:fd:f3 
INFO adding MAC interface map to host static network config - Name:  enp2s0  MacAddress: 52:54:00:e8:b9:18 
INFO[0000] Adding NMConnection file <enp2s0 .nmconnection="">  pkg=manifests
INFO[0000] Adding NMConnection file <enp2s0 .nmconnection="">  pkg=manifests
INFO[0001] Adding NMConnection file <enp2s0 .nmconnection="">  pkg=manifests
INFO[0001] Start configuring static network for 3 hosts  pkg=manifests
INFO[0001] Adding NMConnection file <enp2s0 .nmconnection="">  pkg=manifests
INFO[0001] Adding NMConnection file <enp2s0 .nmconnection="">  pkg=manifests
INFO[0001] Adding NMConnection file <enp2s0 .nmconnection="">  pkg=manifests
INFO Obtaining RHCOS image file from 'https://rhcos-redirector.apps.art.xq1c.p1.openshiftapps.com/art/storage/releases/rhcos-4.11/411.85.202203181601-0/x86_64/rhcos-411.85.202203181601-0-live.x86_64.iso' 
INFO  
~~~

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

During the boot process the system will come up to a standard login prompt on the console.  Then in the background on the host it will start pulling in the required containers to run the familiar Assisted Installer UI.  I gave this process about 5 minutes before I attempted to access the web UI.   To access the web UI we can point our browser to the ipaddress of node we just booted and port 8080:

<img src="ai-boot.png" style="width: 800px;" border=0/>


