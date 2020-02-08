# Acro Media NFS Server/Client Ansible Roles

This role contains 4 mini roles that all work together (see example playbook below):
* **acromedia.nfs/access**: Creates users / groups on clients + server
* **acromedia.nfs/client**: Installs software on your client node(s)
* **acromedia.nfs/server**: Installs software on your NFS server
* **acromedia.nfs/shares**: Creates mount point(s) + share(s) on your server, and mounts the share to your clients


## Use case

This is for when you want to serve a central uploads directory (i.e Drupal's sites/default/files dir) from one central NFS node, and mount it from all of your application nodes.


### A warning about security

NFS has virtually **no** built in security. Make sure you configure your firewall to only allow access from fully trusted systems. Don't serve NFS from any system connected directly to the internet.


## Requirements

- **Your playbook must gather facts.**

- **OS**: Ubuntu >= 16.04, or Red Hat / CentOS 6

- **Firewall**: The following ports need to be open on your NFS server, but ***ONLY*** for access by your app nodes:
    - 111 (tcp) - port mapper service
    - 2049 (tcp) - nfs service
    - If using the `nfs_mountd_port` option (see example below), then also the TCP port specified there. Mount.d normally uses a dynamic port, but this role can optionally wire it to a single port if necessary.

- **Share / mount paths**:
  - The server role creates the path to the shared directory on the NFS server if it doesn't exist.
  - The client role does NOT create the path to the mount point on the client side, since the correct permissions + ownership leading to your mount point may not be trivial. This can be a catch 22 if the nfs client role exists in your playbook before any virtual host directory structure is created. The easiest way around this is to plan on running run your app node playbook twice. The client role creates the necessary fstab entry and mounts the share once the mount point path exists.
  - When creating an NFS share on Red Hat, keep an eye out for overly restrictive home directory permissions. If your nfs host user directory was created with mode 0700, the client wont' be able to mount it until you open up the mode on the directory to at least 0711 so NFS can traverse into it.

- **Domain**: All of your machines must use the same domain name for their internal hostnames. For example if your NFS host server's full hostname is foo.exampledomain, then all the connecting client hosts must also be on the exampledomain domain, otherwise your mounted files will end up with "nobody:nobody" ownership, no matter who owns the files on the NFS host. Hint: Keep an eye on your syslog for messages from 'idmap'.


## User and Group configuration

For NFS to work, linux User and Group IDs must be identical on all of the client and server nodes. As long as you haven't created any users for your web application yet, this will be no problem.

If you have configured web account users, you need to go look and and see if they've all got the same UID/GIDs. If they are, great.  Proceed as in the examples. If not, the you need to do one of the following:
  - Scrub your servers and start over from scratch, running these roles first
  - Fix the uids/gids on all your nodes, and then go and fix ownership everywhere (its possible, but a PITA)
  - Define a new set users/groups that you can specify UIDs/GIDs for that are unused.

## Example playbook

`group_vars/all.yml`:
```yaml
---
# Variables required by the "nfs-access" role, used by both playbooks:
nfs_users:
    # bigcorp is the user that owns the non-writeable files on our web server.
  - name: bigcorp
    gid: 1003
    system: false

    # bigcorp-srv is what our PHP FPM process runs as.
  - name: bigcorp-srv
    gid: 999
    system: true

nfs_groups:
  - name: bigcorp
    uid: 1003
    system: false

  - name: bigcorp-srv
    uid: 999
    system: true

nfs_share_dir: /var/www/bigcorp/wwwroot/sites/default/files

# In our case, it's necessary for our main user to also be part of the group that writes the files to the share.
nfs_secondary_groups:
  - user: bigcorp
    secondary_group: bigcorp-srv

```

`file_server.yml`:
```yaml
---
- hosts: file_server_node
  gather_facts: true
  become: true
  roles:  
    - name: Configure users + groups for NFS -- UIDs/GIDs must be identical to those on the client machines
      role: acromedia.nfs/access
      # nfs_groups: See group_vars/all.yml

    - name: Set up the NFS share(s)
      role: acromedia.nfs/shares
      vars:
        nfs_shares:
          - path: "{{ nfs_share_dir }}"
            owner: bigcorp
            group: bigcorp-srv
            mode: "2775"
            allow_from:
              - '10.0.0.0/24'

    - name: Install the NFS service
      role: acromedia.nfs/server
      vars:
        # Optional: Force mount.d to listen on a single port, instead of letting it be dynamic.
        # It makes firewall configuration simpler & safer.
        nfs_mountd_port: 33333

```

`app_nodes.yml`:
```yaml
---
- hosts: app_nodes
  gather_facts: true
  become: true
  roles:  
    - name: Configure users + groups for NFS
      role: acromedia.nfs/access
      # nfs_groups: See group_vars/all.yml

    - name: Install NFS client software & create the fstab entry.
      role: acromedia.nfs/client
      vars:
        nfs_client_share_mount_point: "{{ nfs_share_dir }}"
        nfs_host: 10.0.0.50
```
