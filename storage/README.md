# Storage

OpenShift abstracts storage, and it's up to the administrator to setup/configure/manage storage. Here is info, again, in no paticular order

* [Host Path](#host-path)
* [NFS](#nfs)

CNS (Container Native Storage), is a whole other beast. Notes for that can be found [here](../cns)

## Host Path

If you are going to add `hostPath` for your application, then you might need to do the following

```
oc edit scc privileged
```

And add under users
```
- system:serviceaccount:default:registry
- system:serviceaccount:default:docker
```

Maybe this will work too? (prefered

```
oc adm policy add-scc-to-user privileged -z registry
oc adm policy add-scc-to-user privileged -z router
```

If you're using `/registry` as your registry storage...

```
semanage fcontext -a -t svirt_sandbox_file_t "/registry(/.*)?"
restorecon -vR /registry
```

Or

```
chcon -R -t svirt_sandbox_file_t /registry
```

## NFS

NFS is a supported protocol and the most common.

* [Setting Up NFS](#setting-up-nfs)
* [NFS Master Config](#nfs-master-config)
* [NFS Client Config](#nfs-client-config)

### Setting Up NFS

Installing/setting up NFS is beyond the scope of this paticular doc; but I do have notes on how to install an NFS server

* [Linux NFS Server Setup](https://github.com/christianh814/notes/blob/master/documents/nfs_notes.md#nfs-v4)
* [Ansible Config](https://github.com/christianh814/notes/blob/master/documents/nfs_notes.md#nfs-v4)

#### Ansible Config

You can use the ansible installer to install NFS for you.

1. Set up your `[OSEv3:children]` to include an `nfs` option. It'll look like this.

```
[OSEv3:children]
masters
nodes
etcd
```

2. Then add an `[nfs]` section with the nfs server's hostname/ip

```
[nfs]
nfs.example.com
```

3. In your `[OSEv3:vars]` section; you can set up your registry, etc to use NFS

```
# NFS Host Group
# An NFS volume will be created with path "nfs_directory/volume_name"
# on the host within the [nfs] host group.  For example, the volume
# path using these options would be "/exports/registry"
openshift_hosted_registry_storage_kind=nfs
openshift_hosted_registry_storage_access_modes=['ReadWriteMany']
openshift_hosted_registry_storage_nfs_directory=/exports
openshift_hosted_registry_storage_nfs_options='*(rw,root_squash)'
openshift_hosted_registry_storage_volume_name=registry
openshift_hosted_registry_storage_volume_size=50Gi
```
### NFS Master Config

First (after creating/exporting storage on the NFS server), create the PV (persistant volume) definition

```
{
  "apiVersion": "v1",
  "kind": "PersistentVolume",
  "metadata": {
    "name": "pv0001"
  },
  "spec": {
    "capacity": {
        "storage": "20Gi"
        },
    "accessModes": [ "ReadWriteMany" ],
    "nfs": {
        "path": "/var/export/vol1",
        "server": "nfs.example.com"
    }
  }
}
```


Create this object as the administrative user

```
root@master# oc login -u system:admin
root@master# oc create -f pv0001.json
persistentvolumes/pv0001
```

This defines a volume for OpenShift projects to use in deployments. The storage should correspond to how much is actually available (make each volume a separate filesystem if you want to enforce this limit). Take a look at it now:

```
root@master# oc describe persistentvolumes/pv0001
Name:		pv0001
Labels:		<none>
Status:		Available
Claim:
Reclaim Policy:	%!d(api.PersistentVolumeReclaimPolicy=Retain)
Message:	%!d(string=)
```

### NFS Client Config

Now on the client side...

Before you add the PV make sure you allow containers to mount NFS volumes

```
root@master# setsebool -P virt_use_nfs=true
root@node1#  setsebool -P virt_use_nfs=true
root@node2#  setsebool -P virt_use_nfs=true
```

Now that the administrator has provided a PersistentVolume, any project can make a claim on that storage. We do this by creating a PersistentVolumeClaim (pvc) that specifies what kind and how much storage is desired:

```
{
  "apiVersion": "v1",
  "kind": "PersistentVolumeClaim",
  "metadata": {
    "name": "claim1"
  },
  "spec": {
    "accessModes": [ "ReadWriteMany" ],
    "resources": {
      "requests": {
        "storage": "20Gi"
      }
    }
  }
}
```

We can have alice do this in the project you created (note accessmodes/storage-size must match):

```
user@host$ oc login -u alice
user@host$ oc create -f pvclaim.json
persistentvolumeclaims/claim1
```

This claim will be bound to a suitable PersistentVolume (one that is big enough and allows the requested accessModes). The user does not have any real visibility into PersistentVolumes, including whether the backing storage is NFS or something else; they simply know when their claim has been filled ("bound" to a PersistentVolume).

```
user@host$ oc get pvc
NAME      LABELS    STATUS    VOLUME
claim1    map[]     Bound     pv0001
```

Finally, we need to modify the DeploymentConfig to specify that this volume should be mounted

```
oc volumes dc/gogs --add --claim-name=gogs-repos-claim --mount-path=/home/gogs/gogs-repositories -t persistentVolumeClaim
oc volumes dc/gogs-postgresql --add --name=pgsql-data --claim-name=pgsql-claim --mount-path=/var/lib/pgsql/data -t persistentVolumeClaim --overwrite
```

Take special note that you're overwriting the right `--name`. Find out with `oc volume dc <myapp> --list`
