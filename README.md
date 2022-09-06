# **Meet The New Agent-Based OpenShift Installer**

<img src="agent-based.png" style="width: 1000px;" border=0/>

With OpenShift 4.11 we are introducing a pre-release version of the [new agent-based installer for OpenShift](https://github.com/openshift/installer/tree/agent-installer) that we are adding to the official OpenShift installer. With this new <code>agent</code> subcommand, installing clusters on premise has never been so easy, while allowing as many types of designs as possible. Our aim is to provide the flexibility of [user-provided infrastructure (UPI)](https://docs.openshift.com/container-platform/4.11/installing/installing_bare_metal/installing-bare-metal.html) installs with the ease of use that the [OpenShift Assisted Installer](https://docs.openshift.com/container-platform/4.11/installing/installing_on_prem_assisted/installing-on-prem-assisted.html) offers for connected environments, while in fully disconnected or air-gapped environments. 

We have worked with users from different industries and incorporated their feedback and use cases in its design. To do this we are combining existing technologies and experience coming from the Assisted Installer and the [installer-provisioned infrastructure (IPI)](https://docs.openshift.com/container-platform/4.11/installing/installing_bare_metal_ipi/ipi-install-overview.html), among others.

## Use Cases We Are Addressing

* Quickly deploy isolated OpenShift clusters on premise of any topology.
* Install a cluster zero with OpenShift management components such as Red Hat Advanced Cluster Management for Kubernetes (ACM), GitOps, Quay, and components to support your other clusters.
* As we start with Hypershift, install the OpenShift cluster that will house the control planes for all the other clusters.

## Features Included and/or In Progress

* Create bootable images with <code>openshift-install</code> command to deploy your clusters.
* In-place bootstrap, no extra node required.
* Works in fully disconnected deployments.
* Works with a mirrored local registry.
* Supports single node OpenShift (SNO), compact 3-node clusters, and highly available topologies.
* Can be automated with third party tools.
* User-friendly interface based on the Assisted Installer.

## Limitations/Notes

 * Errors in the console: some errors in wait-for can be safely ignored ([AGENT-321](https://issues.redhat.com/browse/AGENT-321)).
 * Conflict when both networkConfig and nmstateconfig are provided for a node ([AGENT-322](https://issues.redhat.com/browse/AGENT-322)).
 * dnsmasq bug halts installation after booting ISO in rare circumstances for SNO clusters. SSHing to node and restarting dnsmasq allows it to continue ([BZ#2120357](https://bugzilla.redhat.com/show_bug.cgi?id=2120357)).
 * Misleading error message when lacking assets to create the installation image ([OCPBUGS-527](https://issues.redhat.com/browse/OCPBUGS-527)).
 * When using static IP and bond interfaces hostnames aren’t set ([BZ#2118817](https://bugzilla.redhat.com/show_bug.cgi?id=2118817)).


## Lab Demo Configuration

Throughout the rest of this blog we will be demonstrating how to use the Agent-Based Installer.  While the purpose of the installer is to address bare metal [disconnected environments](https://docs.openshift.com/container-platform/4.11/installing/disconnected_install/installing-mirroring-disconnected.html) our lab demo configuration will consist of 3 virtual machines on a single KVM Red Hat Enterprise Linux hypervisor.  While they are virtual machines they will still demonstrate the workflow that would be experienced on regular bare metal servers.

## Getting Familiar with Agent-Based OpenShift Installer

Let us familiarize ourselves with the Agent-Based Installer workflow.   Unlike the bare metal IPI OpenShift installation there is no need for a provisioning host.  This is because one of the nodes runs the assisted service early in the boot process and eventually becomes the bootstrap host.  The assisted service validates and confirms all hosts checking in meet the requirements and triggers a cluster deployment.   Once the cluster deployment kicks off all the nodes get their RHCOS image written to disk but only the non-bootstrap nodes reboot and begin to instantiate a cluster.  Once they come up then the original bootstrap node reboots and comes up from disk to join the cluster.  At that point the bootstrapping is complete and the cluster comes up just like any other installation method until finalized.

<img src="node-lifecycle.png" style="width: 1000px;" border=0/>

## Obtaining the Binary

Now that we understand how the Agent-Based Installer works and what use cases its trying to solve lets focus on actually using it.  We can obtain the <code>openshift-install</code> 4.11 dev-preview binary which will contain the agent option from the following [link](https://github.com/openshift/installer/releases/tag/agent-installer-v4.11.0-dev-preview-2).  When the links open the page will look like the following:

<img src="openshift-install-dev-preview.png" style="width: 1000px;" border=0/>

From there we can just click on the link to download the <code>openshift-install</code> binary and then set the execution bits on it.

In the future, once the agent installer code merges with the openshift-install master branch, we will be able to use [4.12 developer preview builds](https://amd64.ocp.releases.ci.openshift.org/#4-dev-preview) and extract the openshift-install command using the commands below:

~~~bash
$ export VERSION="4.12.0-ec.2"
$ export CMD=openshift-install
$ export EXTRACT_DIR=$(pwd)
$ export PULLSECRET=/home/bschmaus/pull-secret.json
$ export RELEASE_IMAGE=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/release.txt| grep 'Pull From: quay.io' | awk -F ' ' '{print $3}' | xargs)
$ oc adm release extract --registry-config "${PULLSECRET}" --command=$CMD --to "${EXTRACT_DIR}" ${RELEASE_IMAGE}

$ ./openshift-install version
./openshift-install 4.12.0-ec.2
built from commit 3f27a2e7bedf151101feaa6f4df92b7e052fcd48
release image quay.io/openshift-release-dev/ocp-release@sha256:c7997bb9f6a971b38c430f24ac960834a2988442a3bfb2d5e7e1e6b0aa23676f
release architecture amd64
~~~

Before we start to use the binary we should also ensure that <code>nmstate</code> is installed on the system we are running the <code>openshift-install</code> command from:

~~~
$ sudo rpm -qa|grep ^nmstate
nmstate-1.2.1-3.el8_6.x86_64
~~~

If <code>nmstate</code> is not installed please use <code>dnf -y install nmstate</code> to install the required package.

## Creating the Install-config.yaml and Agent-config.yaml

Now that we have the binary ready at <code>~/openshift-install</code> lets go ahead and create a directory that will contain the required manifests we need for our deployment that will get injected into the ISO we build:

~~~bash
$ pwd
/home/bschmaus/installer

$ mkdir kni-22
~~~

This creates the directory <code>installer/kni-22</code>. With the directory created we can move onto creating the install configuration resource file.  This file specifies the cluster's configuration such as number of control plane and/or worker nodes, the API and ingress VIP, physical node MAC addresses and the cluster networking. In my example I will be deploying a 3 node compact cluster named kni22.   We will also define the image content source policy and include our cert for our registry since we are doing a disconnected installation.

~~~bash
$ cat << EOF > ./kni-22/install-config.yaml
apiVersion: v1
baseDomain: schmaustech.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: kni22
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.0.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    hosts:
      - name: asus3-vm1
        role: master
        bootMACAddress: 52:54:00:e7:05:72
      - name: asus3-vm2
        role: master
        bootMACAddress: 52:54:00:95:fd:f3
      - name: asus3-vm3
        role: master
        bootMACAddress: 52:54:00:e8:b9:18
    apiVIP: "192.168.0.125"
    ingressVIP: "192.168.0.126"
pullSecret: '{ "auths": { "provisioning.schmaustech.com:5000": {"auth": "ZHVtbXk6ZHVtbXk=","email": "bschmaus@schmaustech.com" } } }'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCoy2/8SC8K+9PDNOqeNady8xck4AgXqQkf0uusYfDJ8IS4pFh178AVkz2sz3GSbU41CMxO6IhyQS4Rga3Ft/VlW6ZAW7icz3mw6IrLRacAAeY1BlfxfupQL/yHjKSZRze9vDjfQ9UDqlHF/II779Kz5yRKYqXsCt+wYESU7DzdPuGgbEKXrwi9GrxuXqbRZOz5994dQW7bHRTwuRmF9KzU7gMtMCah+RskLzE46fc2e4zD1AKaQFaEm4aGbJjQkELfcekrE/VH3i35cBUDacGcUYmUEaco3c/+phkNP4Iblz4AiDcN/TpjlhbU3Mbx8ln6W4aaYIyC4EVMfgvkRVS1xzXcHexs1fox724J07M1nhy+YxvaOYorQLvXMGhcBc9Z2Au2GA5qAr5hr96AHgu3600qeji0nMM/0HoiEVbxNWfkj4kAegbItUEVBAWjjpkncbe5Ph9nF2DsBrrg4TsJIplYQ+lGewzLTm/cZ1DnIMZvTY/Vnimh7qa9aRrpMB0= bschmaus@provisioning'
imageContentSources:
- mirrors:
  - provisioning.schmaustech.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - provisioning.schmaustech.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIGKDCCBBCgAwIBAgIUVu+F6PrAXwxVfPs4D0KQA3+50y4wDQYJKoZIhvcNAQEL
  BQAwgYQxCzAJBgNVBAYTAlVTMRYwFAYDVQQIDA1Ob3J0aENhcm9saW5hMRAwDgYD
  VQQHDAdSYWxlaWdoMRAwDgYDVQQKDAdSZWQgSGF0MRIwEAYDVQQLDAlNYXJrZXRp
  bmcxJTAjBgNVBAMMHHByb3Zpc2lvbmluZy5zY2htYXVzdGVjaC5jb20wHhcNMjIw
  MjE2MjAzNjQwWhcNMjMwMjE2MjAzNjQwWjCBhDELMAkGA1UEBhMCVVMxFjAUBgNV
  BAgMDU5vcnRoQ2Fyb2xpbmExEDAOBgNVBAcMB1JhbGVpZ2gxEDAOBgNVBAoMB1Jl
  ZCBIYXQxEjAQBgNVBAsMCU1hcmtldGluZzElMCMGA1UEAwwccHJvdmlzaW9uaW5n
  LnNjaG1hdXN0ZWNoLmNvbTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIB
  AKfEtYL50jb7DYOMn5gF2PCfnn/ah/m3K/cHXFzu3pjeZ8rlsO9yV+u+TYGmQnMK
  obfKUB5F2twMkGEdX3cHyRvo0YiNVzlS9HEt8A1MejlLBGaNvLXLKCpy5eC1tVwh
  51Nj4pscmACd1WNla+KzYqUURRW90SEEN6jNgoabuip2vXLRZLYqDG8uAZ+fFLnz
  Hz0/pbR0bv1mvA0eppLgvGS7DBQdl+LoNg9ZZj/0CgveCmjMFH+5Dw9hMIbPMEpX
  zrG8DgzgME5jIJYzsMu7aOBmQgappfHoU25RC87/vDfGJI65/bQESo7mdLnSmccz
  ADssp4SVnWPOfWotWNalh7hN64Xp4lbf4Kz2ESvQXU+oOwxhlEZhhJJ8KP7Ei6AL
  4mvXSWgXB69GQHwRJ3bi/AdaQC8LcaIRPm+Toy+UUgfNxtJLyi55lsIYjpig7ndW
  vg9kEQUAWz9pr95hn2l9jodGmAcli2s4UDbvhW464NjaVrf8vttpcgxvL3KZBSEZ
  JjkCiQMy5n+BEDZJeiMGDz4guiRk7QW9WrKw8vhPRhKo4ga8kS4bBd+Y1FXrhfiP
  n5V72sSZbPyi66HXGVJUehsbxynfSebaT0ox29j4JyypvzMgMhUCh0tINHqGhGxy
  Rrm67cy9i20JlOXJ9pIXA8REg9avlH4o0WZ1pC63MTiRAgMBAAGjgY8wgYwwHQYD
  VR0OBBYEFFVYOJ5T/qQL71IYyy9iA9CVME9zMB8GA1UdIwQYMBaAFFVYOJ5T/qQL
  71IYyy9iA9CVME9zMA8GA1UdEwEB/wQFMAMBAf8wJwYDVR0RBCAwHoIccHJvdmlz
  aW9uaW5nLnNjaG1hdXN0ZWNoLmNvbTAQBgNVHSAECTAHMAUGAyoDBDANBgkqhkiG
  9w0BAQsFAAOCAgEAoTmLrWsAA0vTq5ALmDa4HBE+qoygqgU5MxDpWZoObGXmo+TK
  se51NPaMyCt3dMsDY8jeDHchr4tVr272J3jDK6ORhQ5v43qJYgAN9Ilnu165QtqD
  WICnXQvnb8PN0zw0ilm9qB6gSwo+1dvHTR5BJ7B3au7WarpUeC8CclIAzEJjSwEo
  Qtdown9cFyC+bShFvL+jZCe7IuWV/AQDZGodNgTAWX0frUklZBcMyJ3LbazCMpb/
  INvO0D1ZrFt8U1QujCKY+2ba7cxnuNwzrJwugQU9FSYrsGHQCthOH6tNwxhuW7Rh
  UqIFLwS25nrfYiu+9EiXXGvjvymCGAwX4d4vGFAJsdPZgkDa3QuW6dq3IF4hkT9v
  loe/LSt9Dr7l9baqnVgg67qc/99T56uI5tiCu8UjJQpHhDDhoJJRnwOOEdStk9hR
  k8niIe373gQ7RBu/jJ0gt/h8CzKRbuRAIS7BTmvOPUe1pGjWh8ZaWyMdQE9/k8cQ
  hHXB0vjqxw15+DPso7SvxWM2Hzhdj5E/cDlXlQfkEEzk6Qk9+I4WY71qsRdguHZl
  q8YYyrzZ5ZiHHHIFfKDZ4Tr50buqLG1Eh34jukA+La7kdOjQhMF4jjdWHJdg/fpn
  wo0DwOKYVO1M1IUl162LHGJDGUr0RUfriX4pKsdg2LvaxuBeNA6HI3QQO+c=
  -----END CERTIFICATE-----
EOF
~~~

The next configuration file we need to create is the agent configuration resource file.   This file will contain the details of the actual hosts in relation to their networking configuration.   Looking close we can see that for each host we define the interface, mac address, ipaddress if static and a DNS resolver and routes, using the NMState configuration format.  The rendezvousIP is the IP address of the host that will become the temporary bootstrap node while the cluster is installing.  This IP address should match one of the nodes, whether they are using a static IP address or a DHCP reservation IP address:

~~~bash
$ cat << EOF > ./kni-22/agent-config.yaml
apiVersion: v1alpha1
metadata:
  name: kni22
rendezvousIP: 192.168.0.116
hosts:
  - hostname: asus3-vm1
    interfaces:
     - name: enp2s0
       macAddress: 52:54:00:e7:05:72
    networkConfig:
      interfaces:
        - name: enp2s0
          type: ethernet
          state: up
          mac-address: 52:54:00:e7:05:72
          ipv4:
            enabled: true
            address:
              - ip: 192.168.0.116
                prefix-length: 23
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
  - hostname: asus3-vm2
    interfaces:
     - name: enp2s0
       macAddress: 52:54:00:95:fd:f3
    networkConfig:
      interfaces:
        - name: enp2s0
          type: ethernet
          state: up
          mac-address: 52:54:00:95:fd:f3
          ipv4:
            enabled: true
            address:
              - ip: 192.168.0.117
                prefix-length: 23
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
  - hostname: asus3-vm3
    interfaces:
     - name: enp2s0
       macAddress: 52:54:00:e8:b9:18
    networkConfig:
      interfaces:
        - name: enp2s0
          type: ethernet
          state: up
          mac-address: 52:54:00:e8:b9:18
          ipv4:
            enabled: true
            address:
              - ip: 192.168.0.118
                prefix-length: 23
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
EOF
~~~

At this point we have now created two configuration files: the install-config.yaml and the agent-config.yaml.  Both of which are under the kni-22 directory:

~~~bash
$ ls -1 kni-22/
agent-config.yaml
install-config.yaml
~~~

## Building the Agent.iso Image

We are now ready to use the Openshift install binary we compiled earlier with the Agent Installer code to generate our ephemeral OpenShift ISO.   We do this by issuing the following command which introduces the agent option.  This in turn will read in the manifest details we generated and download the corresponding RHCOS image and then inject our details into the image writing out a file called agent.iso:

~~~bash
$ bin/openshift-install agent create image --dir kni-22
INFO[0000] Start configuring static network for 3 hosts  pkg=manifests
INFO[0000] Adding NMConnection file <enp2s0.nmconnection>  pkg=manifests
INFO[0001] Adding NMConnection file <enp2s0.nmconnection>  pkg=manifests
INFO[0001] Adding NMConnection file <enp2s0.nmconnection>  pkg=manifests
INFO[0001] Extracting base ISO from release payload     
INFO Consuming Install Config from target directory 
INFO Consuming Agent Config from target directory 
~~~

Once the agent create image command completes we are left with a agent.iso image and an auth directory that containers kubeconfig:

~~~bash
$ ls -1 kni-22/
agent.iso
auth
$ ls -1 kni-22/auth/
kubeconfig
~~~

## Staging the Image on Nodes

Since the nodes I will be using to demonstrate this 3 node compact cluster are virtual machines all on the same KVM hypervisor I will go ahead and copy the agent.iso image over to that host:

~~~bash
$ scp ./kni-22/agent.iso root@192.168.0.22:/var/lib/libvirt/images/
root@192.168.0.22's password: 
agent.iso 
~~~

With the image moved over, I logged into the hypervisor host and ensured each virtual machine we are using (asus3-vm[1-3]) has the image set.  Further the hosts are designed boot off the ISO if the disk is blank.  We can confirm everything is ready with the following output:

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

## Running and Watching the Deployment

Now lets go ahead and start the virtual machines all at the same time from the hypervisor host:

~~~bash
# virsh start asus3-vm1
Domain asus3-vm1 started

# virsh start asus3-vm2
Domain asus3-vm1 started

# virsh start asus3-vm3
Domain asus3-vm1 started
~~~

If we switch over to the console of one of the virtual machines using virt-manager we should see the CoreOSLive boot screen:

<img src="asus3-vm1-console.png" style="width: 800px;" border=0/>

With the nodes booting we can return to the installer directory where we built the agent image and watch the installation continue. To do this we need to first export the <code>kubeconfig</code> file and then issue the command <code>openshift-install agent</code> with the <code>wait-for install-complete</code> switch:

~~~bash
$ pwd
~/installer

$ export KUBECONFIG=/home/bschmaus/installer/kni-22/auth/kubeconfig

$ bin/openshift-install agent wait-for install-complete --dir kni-22
INFO Waiting for cluster install to initialize. Sleeping for 30 seconds 
INFO Waiting for cluster install to initialize. Sleeping for 30 seconds 
INFO Waiting for cluster install to initialize. Sleeping for 30 seconds 
INFO Checking for validation failures ---------------------------------------------- 
ERROR Validation failure found for asus3-vm1        category=network label=Belongs to majority connected group message=No connectivity to the majority of hosts in the cluster
ERROR Validation failure found for asus3-vm1        category=network label=DNS wildcard not configured message=Parse error for domain name resolutions result
ERROR Validation failure found for asus3-vm1        category=network label=NTP synchronization message=Host couldn't synchronize with any NTP server
ERROR Validation failure found for asus3-vm2        category=network label=Belongs to majority connected group message=No connectivity to the majority of hosts in the cluster
ERROR Validation failure found for asus3-vm2        category=network label=DNS wildcard not configured message=Parse error for domain name resolutions result
ERROR Validation failure found for asus3-vm2        category=network label=NTP synchronization message=Host couldn't synchronize with any NTP server
ERROR Validation failure found for asus3-vm3        category=network label=Belongs to majority connected group message=No connectivity to the majority of hosts in the cluster
ERROR Validation failure found for asus3-vm3        category=network label=DNS wildcard not configured message=Parse error for domain name resolutions result
ERROR Validation failure found for asus3-vm3        category=network label=NTP synchronization message=Host couldn't synchronize with any NTP server
ERROR Validation failure found for cluster          category=hosts-data label=all-hosts-are-ready-to-install message=The cluster has hosts that are not ready to install.
INFO Checking for validation failures ---------------------------------------------- 
ERROR Validation failure found for asus3-vm1        category=network label=Belongs to majority connected group message=No connectivity to the majority of hosts in the cluster
ERROR Validation failure found for asus3-vm2        category=network label=Belongs to majority connected group message=No connectivity to the majority of hosts in the cluster
ERROR Validation failure found for asus3-vm3        category=network label=Belongs to majority connected group message=No connectivity to the majority of hosts in the cluster
ERROR Validation failure found for cluster          category=hosts-data label=all-hosts-are-ready-to-install message=The cluster has hosts that are not ready to install.
INFO Checking for validation failures ---------------------------------------------- 
ERROR Validation failure found for cluster          category=hosts-data label=all-hosts-are-ready-to-install message=The cluster has hosts that are not ready to install.
INFO Pre-installation validations are OK          
INFO Cluster is ready for install                 
INFO Host asus3-vm1: updated status from insufficient to known (Host is ready to be installed) 
INFO Preparing cluster for installation           
INFO Host asus3-vm3: updated status from known to preparing-for-installation (Host finished successfully to prepare for installation) 
INFO Host asus3-vm1: New image status quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:b6f2a69fdc1a0844565320fc51316aa79ad6d4661326b30fa606123476c3d9f7. result: success. time: 0.83 seconds; size: 378.98 Megabytes; download rate: 479.70 MBps 
INFO Host asus3-vm1: updated status from preparing-for-installation to preparing-successful (Host finished successfully to prepare for installation) 
INFO Cluster installation in progress             
INFO Host asus3-vm3: updated status from preparing-successful to installing (Installation is in progress) 
INFO Host: asus3-vm2, reached installation stage Writing image to disk 
INFO Host: asus3-vm1, reached installation stage Writing image to disk 
INFO Host: asus3-vm2, reached installation stage Writing image to disk: 24% 
INFO Host: asus3-vm2, reached installation stage Writing image to disk: 30% 
INFO Host: asus3-vm3, reached installation stage Writing image to disk: 45% 
INFO Host: asus3-vm1, reached installation stage Writing image to disk: 43% 
INFO Host: asus3-vm1, reached installation stage Writing image to disk: 57% 
INFO Host: asus3-vm1, reached installation stage Writing image to disk: 62% 
INFO Host: asus3-vm1, reached installation stage Writing image to disk: 72% 
INFO Host: asus3-vm1, reached installation stage Writing image to disk: 77% 
INFO Host: asus3-vm1, reached installation stage Writing image to disk: 85% 
INFO Host: asus3-vm2, reached installation stage Writing image to disk: 100% 
INFO Host: asus3-vm2, reached installation stage Rebooting 
INFO Host: asus3-vm1, reached installation stage Waiting for control plane: Waiting for masters to join bootstrap control plane 
INFO Cluster Kube API Initialized                 
INFO Host: asus3-vm2, reached installation stage Configuring 
INFO Host: asus3-vm2, reached installation stage Joined 
INFO Host: asus3-vm1, reached installation stage Waiting for bootkube 
INFO Host: asus3-vm3, reached installation stage Done 
INFO Host: asus3-vm1, reached installation stage Waiting for controller: waiting for controller pod ready event 
INFO Bootstrap configMap status is complete       
INFO cluster bootstrap is complete                
INFO Cluster is installed                         
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 
INFO     export KUBECONFIG=/home/bschmaus/installer/kni-22/auth/kubeconfig 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.kni22.schmaustech.com 
~~~

Once the cluster installation has completed we can run a few commands to validate that indeed the cluster is up and operational:

~~~bash
$ oc get nodes
NAME        STATUS   ROLES           AGE   VERSION
asus3-vm1   Ready    master,worker   16m   v1.24.0+9546431
asus3-vm2   Ready    master,worker   30m   v1.24.0+9546431
asus3-vm3   Ready    master,worker   30m   v1.24.0+9546431

$ oc get co
NAME                                       VERSION       AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.11.0        True        False         False      4m38s   
baremetal                                  4.11.0        True        False         False      28m     
cloud-controller-manager                   4.11.0        True        False         False      30m     
cloud-credential                           4.11.0        True        False         False      33m     
cluster-autoscaler                         4.11.0        True        False         False      28m     
config-operator                            4.11.0        True        False         False      29m     
console                                    4.11.0        True        False         False      11m     
csi-snapshot-controller                    4.11.0        True        False         False      29m     
dns                                        4.11.0        True        False         False      29m     
etcd                                       4.11.0        True        False         False      27m     
image-registry                             4.11.0        True        False         False      14m     
ingress                                    4.11.0        True        False         False      20m     
insights                                   4.11.0        True        False         False      52s     
kube-apiserver                             4.11.0        True        False         False      25m     
kube-controller-manager                    4.11.0        True        False         False      26m     
kube-scheduler                             4.11.0        True        False         False      25m     
kube-storage-version-migrator              4.11.0        True        False         False      29m     
machine-api                                4.11.0        True        False         False      25m     
machine-approver                           4.11.0        True        False         False      28m     
machine-config                             4.11.0        True        False         False      28m     
marketplace                                4.11.0        True        False         False      29m     
monitoring                                 4.11.0        True        False         False      13m     
network                                    4.11.0        True        False         False      30m     
node-tuning                                4.11.0        True        False         False      28m     
openshift-apiserver                        4.11.0        True        False         False      14m     
openshift-controller-manager               4.11.0        True        False         False      25m     
openshift-samples                          4.11.0        True        False         False      20m     
operator-lifecycle-manager                 4.11.0        True        False         False      29m     
operator-lifecycle-manager-catalog         4.11.0        True        False         False      29m     
operator-lifecycle-manager-packageserver   4.11.0        True        False         False      23m     
service-ca                                 4.11.0        True        False         False      29m     
storage                                    4.11.0        True        False         False      29m     
~~~

## Setting KubeAdmin Password

Now one thing we noticed is that the kubeadmin password is not available from the log output.   This is because this development preview of Agent-Based Installer does not provide that yet.  However we can go ahead and reset the kubeadmin password with the following procedure which will generate a password, set it in kube-system and then write out the password in auth/kubeadmin-password all in one line: 

~~~bash
$ oc patch secret -n kube-system kubeadmin --type json -p '[{"op": "replace", "path": "/data/kubeadmin", "value": "'"$(openssl rand -base64 18 | tee auth/kubeadmin-password | htpasswd -nBi -C 10 "" | cut -d: -f2 | base64 -w 0 -)"'"}]'
secret/kubeadmin patched

$ ls -l auth/kubeadmin-password 
-rw-r-----. 1 bschmaus bschmaus 25 Aug 18 10:50 auth/kubeadmin-password

$ cat auth/kubeadmin-password
VdCOu3+DVfE3J2jurERXFxf1
~~~

And now we should be able to login to the OpenShift Console with our newly reset kubeadmin password.

## Final Thoughts and Stay in Touch

Hopefully this blog provides an early glimpse into the the ease of installation of disconnected clusters on bare metal when using the new Agent-Based Installer.  In future releases when it goes GA we plan to provide additional improvements and enhancements, such as a graphical user interface.

Please, comment or send us feedback either through your usual Red Hat contacts or logging an issue on the [OpenShift installer on Github](https://github.com/openshift/installer/issues).
