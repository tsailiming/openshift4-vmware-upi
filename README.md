# Introduction

This repository contains a collection of Ansible playbooks to help to install Red Hat OpenShift Container Platform 4 on VMware using the UPI method. It currently supports a connected/disconnected environment. No DHCP/PXE is required. 

# Playbooks

|Name  | Description | 
|---|---|
| network_check  | This checks the network, dns and various connectivity of the installation environment.   |  
| bastion_setup | This prepare the bastion server for OCP installation, including seting up a haproxy or mirrored registry.  |   
| lb_check_setup  |  This installs new vms for the load balancer check. | 
| lb_check| This will run checks against the load balancer to ensure it is configured with the correct backends and SSL passthrough is correct. |
| create_iso| This creates a boot iso for each node. |
| ocp_setup | This creates the installer and boot each vm with the iso.|
| destroy | This destroys the OCP vms, excluding the bastion. |
| remove_cdrom | This ejects the CDROM from the OCP nodes. |

# Prerequisite Setup

## Preparing a RHEL 7 template for bastion and Load Balancer check

``` bash
# /bin/sed -i '/^(HWADDR|UUID)=/d' /etc/sysconfig/network-scripts/ifcfg-e*
# yum install -y rsync perl open-vm-tools
# systemctl enable vmtoolsd
```

Then export this vm into a VMware OVA file. 

## Bringing your own repos 

You will need to bring entire directory `/root/repos` to the target environment. 

### Pip
``` bash
# yum localinstall -y https://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/p/python2-pip-8.1.2-12.el7.noarch.rpm
# mkdir -p /root/repos/pip
# (cd /rept/reppos/pip && pip download passlib cryptography pyvmomi bcrypt dnspython netaddr jmespath --no-cache-dir)
```

### RPMS
``` bash
# yum install -y yum-utils
# reposync -n -p /root/repos --repoid rhel-7-server-rpms --repoid rhel-7-server-ansible-2-rpms --repoid rhel-7-server-extras-rpms
```

### OpenShift images for base installation

After perfoming an OC [mirror](https://docs.openshift.com/container-platform/4.3/installing/install_config/installing-restricted-networks-preparations.html#installation-mirror-repository_installing-restricted-networks-preparations)

```  bash
# (cd /opt/registry/data && tar cvzf /root/repos/registry_data.tar.gz .)
```

### Export this git repository
```  bash
# git archive --format=tar.gz --prefix=openshift4-vmware-upi/ master > /root/repos/git.tar.gz
```

### Copy useful binaries

https://stedolan.github.io/jq/download/

https://github.com/vmware/govmomi/releases

```  bash
cp /path/to/jq /root/repos/bin/jq
cp /path/to/govc /root/repos/bin/govc
```

### Prepare registy docker image
``` bash
# yum install -y podman
# podman pull docker.io/library/registry:2
# podman save docker.io/library/registry:2 -o /root/repos/registry.tar
```

### OpenShift installer
``` bash
# cd /root/repos
# curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.2.10/openshift-install-linux-4.2.10.tar.gz
# curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.2.10/openshift-client-linux-4.2.10.tar.gz
# curl -O https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.2/4.2.0/rhcos-4.2.0-x86_64-metal-bios.raw.gz
# curl -O https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.2/4.2.0/rhcos-4.2.0-x86_64-installer.iso
```
# Setup

## Configuring the bastion host
After you have provisioned the bastion host, ensure the fqdn, network and `/etc/resolv.conf` configurations is correct.

Copy /root/repos into bastion:
``` bash
cp -vr /mnt/repos /root/ 
```

If the system can be registered to Satellite, register and enable the following repos:
``` bash
# subscription-manager repos --disable=*
# subscription-manager repos --enable=rhel-7-server-rpms
# subscription-manager repos --enable=rhel-7-server-ansible-2-rpms
# subscription-manager repos --enable=rhel-7-server-extras-rpms
```

Copy binaries:
``` bash
cp /root/repos/bin/{govc,jq} /usr/local/bin/
```

Bootstrap packages. If there is no Satellite, `yum localinstall` from the local `/root/repos` repositories.
``` bash
# yum install ansible rhel-system-roles
```

Install python pip:
``` bash
# yum localinstall -y /root/repos/epel/7/x86_64/Packages/p/python2-pip-8.1.2-10.el7.noarch.rpm
```

Untar Ansible playbooks
``` bash
# (cd /root && tar xvzf /root/repos/git.tar.gz)
```

## Configure Ansible inventory 

The default inventory file is `inventory.yml`

### Ansible host groups

|Name| Description|
|-|-|
|`bastion_grp` | This is the bastion node |
|`apps_lb` | This defines which is the apps load balancer| 
|`master_lb` | This defines which is the masters load balancer| 
|`bootstrap_grp` | This is the boostrap node|
|`masters_grp` | This is a group of masters|
|`workers_grp` | This is a group of workers. All non-masters nodes are part of this group. |
|`infra_routers_grp` | This is a group of infra routers. |

### VM specifications

You can define the VM size by specifying the following host vars. 
``` yaml
ansible_host: xxx.xxx.xxx.xxx
vm_memory_mb: 7168
vm_cpu: 4            
vm_disks: 
  - size: 80 
  	type: thin
```

### Important Ansible variables

| Name | Description|
| - | - |
| setup_haproxy| Whether to configure haproxy on bastion for apps and masters |
| setup_registry| Whether to configure a registry on bastion |
| cluster_name| OCP cluster name|
| base_domain| OCP base domain name|
| openshift_cluster_network_cidr | OpenShift Cluster network CIDR|
| openshift_host_prefix | OpenShift Host Prefix| 
| openshift_service_network_cidr | OpenShift Servie Network CIDR| 
| apps_use_wildcard_dns | Whether to check for wildcard DNS |
| timesync_ntp_servers | ntp servers to configure |
| vm_template | RHEL 7 vm template name | 
| restricted_network | Whether this is restricted network installation |
| yum_repos | If there is no Satallite, configure the local yum repos | 
| yum_conf | If there is no Satallite, configure yum to point to the local repository |
| use_vcp | Whether to integrate OCP with VMware Clod Provider | 

### Super secretive vault.yml in the playbook directory

Create a vault with the following vars:

``` yaml
vcenter_hostname: 
vcenter_username: 
vcenter_password: 
vcenter_insecure_ssl: true

vcp_username: 
vcp_password: 

registry_username: openshift
registry_password: password

ocp_pull_secret: # from cloud.redhat.com when doing a connected install
```

# Running the playbooks

To do a network check. This will create a `/tmp/dns_check.txt` output. It is **recommended** to run this check first. 
``` bash
# ansible-playbook --ask-vault-pass network_check.yml
```

After the network check is successful, it is time to setup the bastion host. Depending on the defined variables, it can optionally setup a HAProxy and Registry for you. 
``` bash
# ansible-playbook --ask-vault-pass bastion_setup.yml
```

We should perform a load balancer check. This creates a `/tmp/lb_check.txt` output file.
``` bash
# ansible-playbook --ask-vault-pass lb_check_setup.yml
# ansible-playbook --ask-vault-pass lb_check.yml

```

After everything is verified to be correct, you can then create the boot isos. This will upload the isos to the datastore defined in the inventory file. 
``` bash
# ansible-playbook --ask-vault-pass create_iso.yml

```

Once the isos are uploaded, we now can create the OpenShift manifest files and virtual machines. The virtual machines will be created and powered on automatically.
``` bash
# ansible-playbook --ask-vault-pass ocp_setup.yml
```

The installation will start and you can continue by following the [OpenShift Installation Guide](https://docs.openshift.com/container-platform/4.3/installing/installing_vsphere/installing-vsphere.html#installation-installing-bare-metal_installing-vsphere). When bootstrap is complete, you can shutdown the bootstrap node.

After installation is completed, you can eject all the cdroms:
``` bash
# ansible-playbook --ask-vault-pass remove_cdrom.yml
```

# Clean Up

To destroy all the vms, excluding bastion:
``` bash
# ansible-playbook --ask-vault-pass destory.yml
```
