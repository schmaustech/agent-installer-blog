# **Meet The New Agent-Based OpenShift Installer**

<img src="agent-based.png" style="width: 1000px;" border=0/>

With OpenShift 4.11 we are introducing a pre-release version of the [new agent-based installer for OpenShift](https://github.com/openshift/installer/tree/agent-installer) that we are adding to the official OpenShift installer. With this new <code>agent</code> subcommand, installing clusters on premise has never been so easy, while allowing as many types of designs as possible. Our aim is to provide the flexibility of [user-provided infrastructure (UPI)](https://docs.openshift.com/container-platform/4.11/installing/installing_bare_metal/installing-bare-metal.html) installs with the ease of use that the [OpenShift Assisted Installer](https://docs.openshift.com/container-platform/4.11/installing/installing_on_prem_assisted/installing-on-prem-assisted.html) offers for connected environments, while in fully disconnected or air-gapped environments. 

We have worked with users from different industries and incorporated their feedback and use cases in its design. To do this we are combining existing technologies and experience coming from the Assisted Installer, the [installer-provisioned infrastructure (IPI)](https://docs.openshift.com/container-platform/4.11/installing/installing_bare_metal_ipi/ipi-install-overview.html) among others.

## Use Cases We are Addressing

* Quickly deploy isolated OpenShift clusters on premise of any topology.
* Install a cluster zero with OpenShift management components such as Red Hat Advanced Cluster Management for Kubernetes (ACM), GitOps, Quay, and components to support your other clusters.
* As we start with Hypershift, install the OpenShift cluster that will house the control planes for all the other clusters.

## Features Included and/or In Progress

* Create bootable images with openshift-install to deploy your clusters.
* In-place bootstrap, no extra node required.
* Works in fully disconnected deployments.
* Works with a mirrored local registry.
* Supports single node OpenShift (SNO), compact 3-node clusters, and highly available topologies.
* Can be automated with third party tools.
* User-friendly interface based on the Assisted Installer.

## Getting Familiar with Agent-Based OpenShift Installer

Write something about the workflow describing the image below

<img src="node-lifecycle.png" style="width: 1000px;" border=0/>

Now that we understand how the Agent-Based Installer works and what use cases its trying to solve lets focus on actually using it.   As of this writing the Agent-based Installer needs to be compiled from the forked installer source code.  This requirements will go away once OpenShift 4.12 goes GA at which time the Agent-based Installer will be GA as well.

So lets go ahead and grab the OpenShift installer source code from Github and checkout the specific commit 06703b2ad2fb337efce4c283acdbc5be07370de9:

~~~bash
$ git clone https://github.com/openshift/installer
Cloning into 'installer'...
remote: Enumerating objects: 213302, done.
remote: Counting objects: 100% (67/67), done.
remote: Compressing objects: 100% (55/55), done.
remote: Total 213302 (delta 22), reused 42 (delta 8), pack-reused 213235
Receiving objects: 100% (213302/213302), 881.35 MiB | 11.15 MiB/s, done.
Resolving deltas: 100% (139159/139159), done.
Updating files: 100% (83711/83711), done.

$ cd installer/
$ git checkout 06703b2ad2fb337efce4c283acdbc5be07370de9
Note: switching to '06703b2ad2fb337efce4c283acdbc5be07370de9'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

HEAD is now at 06703b2ad Merge pull request #6201 from rwsu/BUG-2114977

$ git branch
* (HEAD detached at 06703b2ad)
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
if [ -f main.go ]; then path="."; else path=./vendor/`grep _ tools.go|awk '{ print $2 }'|sed 's|"||g'`; fi; \
go build -ldflags "-s -w" -o ../../bin/linux_amd64/terraform-provider-alicloud "$path"; \
zip -1j ../../bin/linux_amd64/terraform-provider-alicloud.zip ../../bin/linux_amd64/terraform-provider-alicloud;
  adding: terraform-provider-alicloud (deflated 81%)
(...)
cd terraform; \
go build -ldflags "-s -w" -o ../bin/linux_amd64/terraform ./vendor/github.com/hashicorp/terraform
make: Leaving directory '/home/bschmaus/installer/terraform'
+ copy_terraform_to_mirror
++ go env GOOS
++ go env GOARCH
+ TARGET_OS_ARCH=linux_amd64
+ rm -rf '/home/bschmaus/installer/pkg/terraform/providers/mirror/*/'
+ find /home/bschmaus/installer/terraform/bin/linux_amd64/ -maxdepth 1 -name 'terraform-provider-*.zip' -exec bash -c '
      providerName="$(basename "$1" | cut -d - -f 3 | cut -d . -f 1)"
      targetOSArch="$2"
      dstDir="${PWD}/pkg/terraform/providers/mirror/openshift/local/$providerName"
      mkdir -p "$dstDir"
      echo "Copying $providerName provider to mirror"
      cp "$1" "$dstDir/terraform-provider-${providerName}_1.0.0_${targetOSArch}.zip"
    ' shell '{}' linux_amd64 ';'
Copying alicloud provider to mirror
Copying aws provider to mirror
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
Copying time provider to mirror
Copying vsphere provider to mirror
Copying vsphereprivate provider to mirror
Copying azureprivatedns provider to mirror
+ mkdir -p /home/bschmaus/installer/pkg/terraform/providers/mirror/terraform/
+ cp /home/bschmaus/installer/terraform/bin/linux_amd64/terraform /home/bschmaus/installer/pkg/terraform/providers/mirror/terraform/
+ MODE=release
++ git rev-parse --verify 'HEAD^{commit}'
+ GIT_COMMIT=06703b2ad2fb337efce4c283acdbc5be07370de9
++ git describe --always --abbrev=40 --dirty
+ GIT_TAG=unreleased-master-6529-g06703b2ad2fb337efce4c283acdbc5be07370de9
+ DEFAULT_ARCH=amd64
+ GOFLAGS=-mod=vendor
+ LDFLAGS=' -X github.com/openshift/installer/pkg/version.Raw=unreleased-master-6529-g06703b2ad2fb337efce4c283acdbc5be07370de9 -X github.com/openshift/installer/pkg/version.Commit=06703b2ad2fb337efce4c283acdbc5be07370de9 -X github.com/openshift/installer/pkg/version.defaultArch=amd64'
+ TAGS=
+ OUTPUT=bin/openshift-install
+ export CGO_ENABLED=0
+ CGO_ENABLED=0
+ case "${MODE}" in
+ LDFLAGS=' -X github.com/openshift/installer/pkg/version.Raw=unreleased-master-6529-g06703b2ad2fb337efce4c283acdbc5be07370de9 -X github.com/openshift/installer/pkg/version.Commit=06703b2ad2fb337efce4c283acdbc5be07370de9 -X github.com/openshift/installer/pkg/version.defaultArch=amd64 -s -w'
+ TAGS=' release'
+ test '' '!=' y
+ GOOS=
+ GOARCH=
+ go generate ./data
writing assets_vfsdata.go
+ echo ' release'
+ grep -q libvirt
+ go build -mod=vendor -ldflags ' -X github.com/openshift/installer/pkg/version.Raw=unreleased-master-6529-g06703b2ad2fb337efce4c283acdbc5be07370de9 -X github.com/openshift/installer/pkg/version.Commit=06703b2ad2fb337efce4c283acdbc5be07370de9 -X github.com/openshift/installer/pkg/version.defaultArch=amd64 -s -w' -tags ' release' -o bin/openshift-install ./cmd/openshift-install
~~~

Now that we have the binary lets go ahead and create a directory that will contain the required manifests we need for our deployment that will get injected into the ISO we build:

~~~bash
$ pwd
~/installer

$ mkdir cluster-manifests
~~~

With the directory created we can move onto creating the install configuration resource file.  This file specifies the clusters configuration such as number of control plane and/or worker nodes, the api and ingress vip, physical node mac addresses and the cluster networking. In my example I will be deploying a 3 node compact cluster which referenced a cluster deployment named kni22.   We will also define the image content source policy and include our cert for our registry since we are doing a disconnected installation.

~~~bash
$ cat << EOF > ./cluster-manifests/install-config.yaml
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
pullSecret: ''{ "auths": { "provisioning.schmaustech.com:5000": {"auth": "ZHVtbXk6ZHVtbXk=","email": "bschmaus@schmaustech.com" } } }''
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCoy2/8SC8K+9PDNOqeNady8xck4AgXqQkf0uusYfDJ8IS4pFh178AVkz2sz3GSbU41CMxO6IhyQS4Rga3Ft/VlW6ZAW7icz3mw6IrLRacAAeY1BlfxfupQL/yHjKSZRze9vDjfQ9UDqlHF/II779Kz5yRKYqXsCt+wYESU7DzdPuGgbEKXrwi9GrxuXqbRZOz5994dQW7bHRTwuRmF9KzU7gMtMCah+RskLzE46fc2e4zD1AKaQFaEm4aGbJjQkELfcekrE/VH3i35cBUDacGcUYmUEaco3c/+phkNP4Iblz4AiDcN/TpjlhbU3Mbx8ln6W4aaYIyC4EVMfgvkRVS1xzXcHexs1fox724J07M1nhy+YxvaOYorQLvXMGhcBc9Z2Au2GA5qAr5hr96AHgu3600qeji0nMM/0HoiEVbxNWfkj4kAegbItUEVBAWjjpkncbe5Ph9nF2DsBrrg4TsJIplYQ+lGewzLTm/cZ1DnIMZvTY/Vnimh7qa9aRrpMB0= bschmaus@provisioning'
imageContentSources:
- mirrors:
  - provisioning.schmaustech.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - provisioning.schmaustech.com:5000/ocp4/openshift4
  source: registry.svc.ci.openshift.org/ocp/release
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

The next configuration file we need to create is the agent configuration resource file.   This file will contain the details of the actual hosts in relation to their networking configuration.   Looking close we can see that for each host we define the interfac, mac address, ipaddress if static and a DNS resolver and routes.   The configuration is very similar to a NMState configuration.  However one item that is a bit different is the rendezvousIP address.   This is the ipaddress of the host that will become the temporary bootstrap node while the cluster is installing.  This ipaddress shoule match one of the other nodes whether they are using a static ipaddress or a dhcp reservation ipaddress:

~~~bash
$ cat << EOF > ./cluster-manifests/agent-config.yaml
kind: AgentConfig
metadata:
  name: kni22
spec:
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

At this point we have now created two configuration file: the install-config.yaml and the agent-config.yaml.  Both of which are under the cluster-manifests directory:

~~~bash
$ ls -1 cluster-manifests/
agent-config.yaml
install-config.yaml
~~~

One more step we need to execute before creating our image is to set our image release override to ensure we are dpeloying the version of OpenShift we want to deploy.  In my example my local disconnected registry contains the following OpenShift release 4.11.0-rc.7.  So we will want to set the override to that version by executing the following:

~~~bash
export VERSION="4.11.0-rc.7"
export OPENSHIFT_RELEASE_IMAGE="registry.svc.ci.openshift.org/ocp/release:$VERSION"
export LOCAL_REG='provisioning.schmaustech.com:5000'
export export LOCAL_REPO='ocp4/openshift4'
export OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=${LOCAL_REG}/${LOCAL_REPO}:${VERSION}
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


