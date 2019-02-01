# Installation

The installation of OpenShift Container Platform (OCP); will be done via ansible. More information can be found using the OpenShift [documentation site](https://docs.openshift.com/container-platform/latest/welcome/index.html).

* [Infrastrucure](#infrastrucure)
* [Host preparation](#host-preparation)
* [Docker Configuration](#docker-configuration)
* [Ansible Installer](#ansible-installer)
* [Running The Playbook](#running-the-playbook)
* [Package Excluder](#package-excluder)
* [Uninstaller](#uninstaller)
* [Cloud Install](#cloud-install)
* [Disconnected Install](#disconnected-install)

## Infrastrucure 

For this installation we have the following

* Wildcard DNS entry like `*.apps.example.com`
* Servers installed with RHEL 7.x (latest RHEL 7 version) with a "minimum" install profile.
* Forward/Reverse DNS is a MUST for master/nodes
* SELinux should be enforcing
* Firewall should be running.
* NetworkManager 1.0 or later
* Masters
  * 4CPU
  * 16GB RAM
  * Disk 0 (Root Drive) - 50GB
  * Disk 1 - 100GB mounted as `/var`
* Nodes
  * 4CPU
  * 16GB RAM
  * Disk 0 (Root Drive) - 50GB
  * Disk 1 - 100GB mounted as `/var`
  * Disk 2 - 500GB Raw/Unformatted (for Container Native Storage)

Here is a diagram of how OCP is layed out

![ocp_diagram](images/osev3.jpg)

## Host preparation

Each host must be registered using RHSM and have an active OCP subscription attached to access the required packages.

On each host, register with RHSM:

```
subscription-manager register --username=${user_name} --password=${password}
```

List the available subscriptions:

```
subscription-manager list --available
```

In the output for the previous command, find the pool ID for an OpenShift Enterprise subscription and attach it:

```
subscription-manager attach --pool=${pool_id}
```

Disable all repositories and enable only the required ones:

```
subscription-manager repos  --disable=*
yum-config-manager --disable \*
subscription-manager repos \
    --enable=rhel-7-server-rpms \
    --enable=rhel-7-server-extras-rpms \
    --enable=rhel-7-server-ose-3.11-rpms \
    --enable=rhel-7-server-ansible-2.6-rpms \
    --enable=rh-gluster-3-client-for-rhel-7-server-rpms
```

Make sure the pre-req pkgs are installed/removed and make sure the system is updated

```
yum -y install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct vim
yum -y update
systemctl reboot
yum -y install openshift-ansible
```

Then install docker when it comes back up. Make sure you're running the version it states in the docs

```
yum -y install docker-1.13.1
docker version
```

## Docker Configuration

Docker storage doesn't need to be configured as we use `overlayfs`.

I moved the `lvm` config section [here](guides/docker-ocp.md) just for historical purposes. (you won't need them though)

## Ansible Installer

On The master host, generate ssh keys to use for ansible press enter to accept the defaults

```
root@master# ssh-keygen
```

Distribue these keys to all hosts (including the master)

```
root@master# for host in ose3-master.example.com \
    ose3-node1.example.com \
    ose3-node2.example.com; \
    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
    done
```

Test passwordless ssh

```
root@master# for host in ose3-master.example.com \
    ose3-node1.example.com \
    ose3-node2.example.com; \
    do ssh $host hostname; \
    done
```

Make a backup of the `/etc/ansible/hosts` file

```
cp /etc/ansible/hosts{,.bak}
```

Next You must create an `/etc/ansible/hosts` file for the playbook to use during the installation

> **NOTE**, for a PoC install, you can't have LESS than a `/21` (i.e. NO `/24`) or an `osm_host_subnet_length` less than 9

Sample Ansible Hosts files
  * [Single Master](https://raw.githubusercontent.com/christianh814/openshift-toolbox/master/ansible_hostfiles/singlemaster)
  * [Multi Master](https://raw.githubusercontent.com/christianh814/openshift-toolbox/master/ansible_hostfiles/multimaster)

With Cloud installations; you need to enable API access to the cloud provider. Below are example entries (NOT whole "hostfiles"; rather what you need to add to the above)
  * [AWS Hostfile Options](https://raw.githubusercontent.com/christianh814/openshift-toolbox/master/ansible_hostfiles/awsinstall)

Sample HAProxy configs if you want to build your own HAProxy server
  * [HAProxy Config](https://raw.githubusercontent.com/christianh814/openshift-toolbox/master/haproxy_config/haproxy.cfg)
  * [HAProxy with Let's Encrypt](https://raw.githubusercontent.com/christianh814/openshift-toolbox/master/haproxy_config/haproxy-letsencrypt.cfg)

If you used let's encrypt, you might find [these crons](../certbot) useful

## Running The Playbook

You can run the playbook (specifying a `-i` if you wrote the hosts file somewhere else) at this point

First run the prereq playbook

```
root@master# ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
```

Now run the installer afterwards

```
root@master# ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```

Once this completes successfully, run `oc get nodes` and you should see "Ready"

```
root@master# oc get nodes
NAME                                          STATUS    ROLES          AGE       VERSION
ip-172-31-19-75.us-west-1.compute.internal    Ready     compute        10d       v1.11.0+d4cacc0
ip-172-31-23-129.us-west-1.compute.internal   Ready     compute        10d       v1.11.0+d4cacc0
ip-172-31-23-47.us-west-1.compute.internal    Ready     compute        10d       v1.11.0+d4cacc0
ip-172-31-28-6.us-west-1.compute.internal     Ready     infra,master   10d       v1.11.0+d4cacc0
```

I also like to see all my pods statuses

```
oc get pods --all-namespaces
```

Label nodes if the installer didn't for whatever reason...

```
oc label node infra1.cloud.chx node-role.kubernetes.io/infra=true
```

## Package Excluder

OpenShift excludes packages during install, you may want to unexclude it at times (you probably never have to; but here's how to in any event)

```
atomic-openshift-excluder [ unexclude | exclude ]
```

## Uninstaller

If you need to "start over", you can uninstall OpenShift with the following playbook...

```
root@master# ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml
```

Note that this may have unintended consequences (like destroying formatted disks, removing config files, etc). Run this only when needed.

## Cloud Install

Here are Notes about cloud based installations

* [AWS Install](../aws_refarch)
* [Azure Install](https://access.redhat.com/documentation/en-us/reference_architectures/2018/html/deploying_and_managing_openshift_3.9_on_azure/index)
* [GCE Install](https://access.redhat.com/documentation/en-us/reference_architectures/2018/html/deploying_and_managing_openshift_3.9_on_google_cloud_platform/)
* [Openstack Install](https://access.redhat.com/documentation/en-us/reference_architectures/2018/html/deploying_and_managing_openshift_3.9_on_red_hat_openstack_platform_10/index)


## Disconnected Install

There are many factors to take into consideration when trying to do a disconnected install. Instructions/notes for that can be found [HERE](guides/disconnected.md)
