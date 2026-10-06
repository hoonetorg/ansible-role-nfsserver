# ansible-role-nfsserver

NFSv4 server on openSUSE Leap: exports from the inventory, NFSv2/v3 switched off, firewalld service `nfs` in the
listed zones (default: `libvirt`, the zone of the libvirt bridges), SELinux export booleans ensured on.
Typical use: a KVM host exports directories to its guests, e.g. the host's `/home/jd` to a guest's `/home/jd`.

1. installs `nfs-kernel-server`
2. checks every export path (must be a directory; `create: true` creates a missing one)
3. `NFS3_SERVER_SUPPORT="no"` in `/etc/sysconfig/nfs` (SUSE's `/usr/etc/nfs.conf` takes `[nfsd] vers3` from it)
   → restart `nfs-server`
4. writes `/etc/exports.d/ansible.exports` (exactly the listed exports; other files in `/etc/exports.d` are not
   touched) → `exportfs -ra`
5. enables and starts `nfs-server.service`
6. firewalld (if running): every service of `firewalld_services` in every zone of `firewalld_zones`, permanent
   and immediate; a missing zone stops the run (zone `libvirt` exists once libvirt is installed - run
   ansible-role-virt first)
7. SELinux (if enabled): `setsebool -P <boolean> on` only for booleans that are off

Only openSUSE is implemented. With `install: false` (default) the role does nothing.

## Variables

```yaml
nfsserver:
  install: true                      # default false
  exports:
    - path: /home/jd
      clients:
        - host: 192.168.122.0/24     # host, network, wildcard (exports(5))
          options: rw,sync,no_subtree_check,no_root_squash,security_label
      # create: true                 # create a missing path (default: a missing path stops the run)
  # nfsv4_only: true                 # default
  # firewalld_zones: [libvirt]       # default
  # firewalld_services: [nfs]        # default; NFSv4 only needs 2049/tcp
  # selinux_booleans: [nfs_export_all_rw, nfs_export_all_ro]   # default
```

rpcbind/rpc-statd are not masked (the host may still mount NFSv3 shares as a client); with only service `nfs`
open they are not reachable from the guests.

`sec=sys` (the default) maps users by uid: the user must have the same uid on server and client (e.g. jd = 1000
on the host and in the guest).

`security_label` (NFS 4.2) passes the server's SELinux labels to SELinux clients.

## Checks

```bash
cat /proc/fs/nfsd/versions                 # -3 +4 +4.1 +4.2
exportfs -v                                # the exports with their effective options
firewall-cmd --zone=libvirt --list-services    # contains nfs
getsebool nfs_export_all_rw nfs_export_all_ro  # on
# in the guest:
mount -t nfs4 <host address on the bridge>:/home/jd /mnt && ls /mnt && umount /mnt
```

## License

Apache-2.0

Created with the help of AI
