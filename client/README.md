# NFS Client sub-role

* See also: [../README.md](../README.md)

## Variables

* See also: [defaults/main.yml](defaults/main.yml)

#### nfs_client_mount_fstype
* Can be one of `nfs` (default), `efs`, or `none`.

#### nfs_client_efsutils_version
* When `nfs_client_mount_fstype` == `efs`, efs-utils will be installed from https://github.com/aws/efs-utils
* Default value: `HEAD`. If a specific value of `nfs_client_efsutils_version` is not specified, the latest version of `efs-utils` from origin/master will be installed. It's recommended that you find the latest tag/release, and specify that version instead, since master may not be stable.
* Once `efs-utils` is installed, the installed version will not change (even if there are new versions available in the repo) unless you change the value of `nfs_client_efsutils_version`.
