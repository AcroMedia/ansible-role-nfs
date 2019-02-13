# Acro Media NFS Server/Client Ansible Roles

* access
* client
* server
* shares


## Use case

This is for when you want to serve a central uploads directory (i.e Drupal's sites/default/files dir) from one central NFS node, and mount it from all of your application nodes.

## Requirements

OS: Ubuntu >= 16.04

### Users / Groups

Linux User and Group IDs must end up being identical on each of the client nodes as well as the server node.

If you've set up any web account users / groups yet, you need to go look and and see if they've all got the same UID/GIDs. If they are, great.  Proceed as in the examples. If not, the you need to do one of the following:
- Scrub your servers and start over from scratch, running these roles first
- Fix the uids/gids on all your nodes, and then go and fix ownership everywhere (its possible, but a PITA)
- Define a new set users/groups that you can specify UIDs/GIDs for that are unused.

### Security

NFS has virtually **no** built in security. Make sure you configure your firewall to only allow access from fully trusted systems. Don't serve NFS from any system connected directly to the internet.

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

    - name: Install the NFS service
      role: acromedia.nfs/server

    - name: Set up the NFS share(s)
      role: acromedia.nfs/shares
      vars:
        nfs_shares:
          - path: "{{ nfs_share_dir }}"
            owner: bigcorp
            group: bigcorp-srv
            mode: 02775
            allow_from: '10.0.0.0/24'
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

    - name: Install NFS client software & Mount point
      role: acromedia.nfs/client
      vars:
        nfs_client_share_mount_point: "{{ nfs_share_dir }}"
        nfs_host: 10.0.0.50
```
