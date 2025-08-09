minio
=========

An ansible role that setup NFS.

Role Variables
--------------

```
# Directories
nfs_export_path: "/mnt/nfs"

# Services
nfs_service_name: "nfs-kernel-server"

# mount
nfs_mount_options: "rw,sync,no_subtree_check,all_squash"

# Server
nfs_network: "10.0.0.0/16"
```

License
-------

`exmaple Limited License`

Author Information
------------------

exmaple - DevOps Department
