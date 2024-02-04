# Booting a Root-on-ZFS NixOS System From Recovery Media

## Goal

Boot a NixOS system that uses [root-on-ZFS](https://openzfs.github.io/openzfs-docs/Getting%20Started/NixOS/Root%20on%20ZFS.html) from NixOS recovery media and trigger a `nixos-rebuild`.


## Why

This is useful to practice in case any of the below happens:

* Recovering from a broken bootloader that means you can't boot your system. The `grub` bootloader can be sensitive to changes in ZFS upstream and some care must be taken that your boot pool is readable by `grub`. Booting from recovery media allows you to reinstall the bootloader.
* Replacing a disk that is failing or recovering from a failed disk.
* Any operation that requires you to rebuild your boot or root ZFS pools, such as recovering from backups or removing native ZFS encryption.


## Process

You'll need physical access to the host or remote hands/BMC (e.g. Intel AMT, HP iLO, Dell iDRAC, pikvm) to access the BIOS and console, at least to boot the image. Once the system is up and running you can use SSH to perform subsequent steps.


### Boot From Media

[Download NixOS live ISO](https://nixos.org/download) (minimal is sufficient) and burn to a USB stick (e.g. using [Balena Etcher](https://etcher.balena.io/)) (or attach image to VM if you're doing this to a virtual instance).

Boot your system using the USB/attached image.

You can do all of the remaining steps directly from the console, but SSH is likely easier:

Set passwd for nixos user:

```
[nixos@nixos:~] passwd
New password:
Retype new password:
passwd: password updated successfully
```

SSH in from another machine:

```
bash-3.2$ ssh nixos@your_host
(nixos@your_host) Password: 
Last login: Fri Jan  5 08:34:14 2024

[nixos@nixos:~]$ 
```

> You might need to temporarily allow connecting to this host if you've connected before as your client will see that the host's SSH server key has changed. Add the `-o StrictHostKeyChecking=no` argument to your `ssh` command.

Become root with `sudo -i`.


## Import ZFS pools

Create a root directory that you will later mount these into:

```
[root@nixos:~]$ mkdir /tmp/root
```

Depending on when you followed the NixOS root-on-ZFS guide and the choices you made when following the guide, you likely ended up with one of the configurations below.


### Import Boot Pool

You likely have a separate ZFS pool for `/boot` (named `bpool` or `bpool_xxxxxx` where `x` are alphanumeric characters) that contains your kernel and initrd. This may be on a LUKS encrypted volume.

> If you're using `grub` bootloader per the root-on-ZFS guide, your boot pool won't be using ZFS native encryption as this is not currently supported by `grub`.

Check your partition table to see if you have a boot pool:

```
[root@nixos:~]# sgdisk -p /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0
Disk /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0: 83886080 sectors, 40.0 GiB
Sector size (logical/physical): 512/4096 bytes
Disk identifier (GUID): 34A6CF43-9D61-4BA7-A83B-D1D31489370C
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 83886046
Partitions will be aligned on 16-sector boundaries
Total free space is 14 sectors (7.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         4196351   2.0 GiB     EF00  
   2         4196352        12584959   4.0 GiB     BE00  
   3        29362176        83886046   26.0 GiB    BF00  
   4        12584960        29362175   8.0 GiB     8200  
   5              48            2047   1000.0 KiB  EF02  
```

If you have a partition with partition code `BE00` you have a ZFS boot pool.

Check if it is encrypted (replace `-part2` with the number of the partition from the above) by seeing if it has a LUKS header:

```
[root@cascade:~]# cryptsetup luksDump /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0-part3 | grep Version
Version:       	2
```

This example above is encrypted with LUKS2.

Example of a non-LUKS partition:

```
[root@cascade:~]# cryptsetup luksDump /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0-part1 | grep Version
Device /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0-part1 is not a valid LUKS device.
```

 (as expected: this partition has type code `EF00` which is EFI)

If encryption is enabled, unlock the volume; otherwise proceed directly to importing the pool. The command below will create a device `/dev/mapper/bpool` that is the unencrypted volume.

```
[root@nixos:~]# cryptsetup open /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0-part2 bpool
Enter passphrase for /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0-part2: 
```

Import the pool:

```
[root@nixos:~]$ zpool import -R /tmp/root -N -f bpool
```

(replace `bpool` with the correct pool name. You can run `zpool import` with no arguments to see what pools are available to import).


### Import your root pool

You likely have a separate root pool (named `rpool` or `rpool_xxxxxx` where `x` are alphanumeric characters) that contains your root filesystem, your Nix store (`/nix`) and persistent state.

This may be stored on an unencrypted volume or a LUKS encrypted volume. It may also use ZFS native encryption.

Identify your partition from the `sgdisk` output above: your root pool uses partition code `BF00` and is probably the largest partition.

As above, check if the pool has LUKS encryption enabled with `cryptsetup luksDump <partition>`. If it does not, proceed to importing the pool. If it does use an encrypted volume, unlock it:

```
[root@nixos:~]# cryptsetup open /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0-part3 bpool
Enter passphrase for /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-0-0-0-part3: 
```


Import the pool:

```
[root@nixos:~]$ zpool import -R /tmp/root -N -f rpool
```

(replace `rpool` with the correct pool name. You can run `zpool import` with no arguments to see what pools are available to import).


Check if your pool uses ZFS native encryption:

```
[root@nixos:~]$ zfs list -o name,encryption,keylocation -d 2 rpool
NAME               ENCRYPTION   KEYLOCATION
rpool              off          none
rpool/nixos        aes-256-gcm  file:///etc/cryptkey.d/rpool_tq0p9b-nixos-key-zfs
rpool/nixos/home   aes-256-gcm  none
rpool/nixos/nix    aes-256-gcm  none
rpool/nixos/root   aes-256-gcm  none
rpool/nixos/state  aes-256-gcm  none
rpool/nixos/var    aes-256-gcm  none
```

If any of the datasets have anything other than `off` for the `ENCRYPTION` column, you will need to load the key before you can mount the dataset(s).

ZFS native encryption currently either supports unlocking via a keyfile or via a passphrase. In the example above, this dataset uses a keyfile. If the `KEYLOCATION` column says `prompt` then your dataset is likely using a passphrase. This is easy to unlock with a command like `zfs load-key rpool/nixos`.

If you need a keyfile to unlock the dataset, the easiest way to get this keyfile is to extract it from the initrd secrets (see NixOS option [`boot.initrd.secrets`](https://search.nixos.org/options?show=boot.initrd.secrets))

Get the location of the latest secrets initrd file:

```
[root@nixos:~]# ls -ltr /boot/kernels/*secrets | tail -3
-rw-r----- 1 root root 260 Feb  3 11:18 /boot/kernels/gaquohleij5ohfi5xugh2iecho9yai0t-nixos-system-wopr-23.11.20231225.12e0c7c-secrets
-rw-r----- 1 root root 260 Feb  3 11:18 /boot/kernels/iethoh7naibei3ahjish5eebeeph8eil-nixos-system-wopr-23.11.20231220.650fa79-secrets
-rw-r----- 1 root root 260 Feb  3 11:18 /boot/kernels/zahgui1fae9oop4ceihagohwaz4ahwah-nixos-system-wopr-23.11.20231220.b45f4a5-secrets
```

Take care with these secrets: they can be used to unlock your disks, so do not store them anywhere that a non-root user can access them or copy them onto an unencrypted volume where they could be read by anyone with physical access to the disk.

Extract the secrets initrd into a temporary directory. On a recovery system, a subdirectory of `/tmp` is fine as it is a ramdisk and its contents will be gone when you reboot your system.

```
[root@nixos:~]# mkdir /tmp/secret

[root@nixos:~]# chmod 0700 /tmp/secret

[root@nixos:~]# mount -t tmpfs nodev /tmp/secret

[root@nixos:~]# zstdcat /boot/kernels/zahgui1fae9oop4ceihagohwaz4ahwah-nixos-system-wopr-23.11.20231220.b45f4a5-secrets | cpio -i -D /tmp/secret
2 blocks

[root@nixos:~]# find /tmp/secret -ls
        1      0 drwxrwxrwt   3 root     root           60 Feb  4 08:20 /temp/secret
        2      0 drwxr-----   3 root     root           60 Feb  4 08:20 /temp/secret/.initrd-secrets
        3      0 drwxr-----   3 root     root           60 Feb  4 08:20 /temp/secret/.initrd-secrets/etc
        4      0 drw-r-----   2 root     root           80 Feb  4 08:20 /temp/secret/.initrd-secrets/etc/cryptkey.d
        5      4 -r--------   1 root     root           32 Feb  4 08:20 /temp/secret/.initrd-secrets/etc/cryptkey.d/bpool_ohv3bi-key-luks
        6      4 -r--------   1 root     root           32 Feb  4 08:20 /temp/secret/.initrd-secrets/etc/cryptkey.d/rpool_ieph1i-nixos-key-zfs

```

You can then unlock your ZFS volume with a command like:

```
[root@thera:~]# zfs load-key -L /temp/secret/.initrd-secrets/etc/cryptkey.d/rpool_ieph1i-nixos-key-zfs rpool/nixos
```



## <a name="mounting-filesystems"></a>Mounting Filesystems

See what's available to mount (your layout might look different but the process should be the same):

```
[root@nixos:~]# zfs list -o space,mountpoint,canmount,mounted
NAME                   AVAIL   USED  USEDSNAP  USEDDS  USEDREFRESERV  USEDCHILD  MOUNTPOINT            CANMOUNT  MOUNTED
rpool                  20.7G  4.01G        0B     96K             0B      4.01G  /tmp/root             off       no
rpool/nixos            20.7G  4.01G        0B     96K             0B      4.01G  none                  off       no
rpool/nixos/home       20.7G  1.31M      128K    124K             0B      1.07M  /tmp/root/home        on        no
rpool/nixos/home/root  20.7G  1.07M      152K    940K             0B         0B  /tmp/root/home/root   on        no
rpool/nixos/nix        20.7G  3.98G     1.16G   2.82G             0B         0B  /tmp/root/nix         on        no
rpool/nixos/root       20.7G  1.04M      480K    584K             0B         0B  /tmp/root             noauto    no
rpool/nixos/state      20.7G  1.15M      184K    992K             0B         0B  /tmp/root/state       on        no
rpool/nixos/var        20.7G  20.6M        0B     96K             0B      20.5M  /tmp/root/var         off       no
rpool/nixos/var/lib    20.7G   480K        0B    480K             0B         0B  /tmp/root/var/lib     on        no
rpool/nixos/var/log    20.7G  20.0M      144K   19.9M             0B         0B  /tmp/root/var/log     on        no
bpool                  1008M  2.64G        0B    176K             0B      2.64G  /tmp/root             off       no
bpool/nixos            1008M  2.63G        0B    176K             0B      2.63G  none                  off       no
bpool/nixos/root       1008M  2.63G     2.33G    304M             0B         0B  /tmp/root/boot        noauto    no
```

Any that have `canmount=noauto` should be done manually first. Mount your system root:

```
[root@nixos:~]# zfs mount rpool/nixos/root
```

Then mount `/boot`:

```
[root@nixos:~]# zfs mount bpool/nixos/root
```

Then mount everything else:

```
[root@nixos:~]# zfs mount -a
```

If you get an error `failed to lock /etc/exports.d/zfs.exports.lock: No such file or directory`, you can ignore it as it relates to writing a file for NFS mounts to the read-only recovery volume which we are not using here.


Mount everything else into the new root:

```
[root@nixos:~]# mount --fstab /tmp/root/etc/fstab --target-prefix /tmp/root --all
```


Set the hostname. This is esential if your system uses flakes for its system configuration, as the hostname in the flake needs to match the current running system for you to successfully build it. For example, if your system is called `wopr`:

```
[root@nixos:~]# hostname wopr
```

Enter the system:

```
[root@wopr:~]# nixos-enter --root /tmp/rpool
setting up /etc...
```

You should be able to now trigger a rebuild in the usual way:

```
[root@wopr:/etc/nixos]# nixos-rebuild switch
building the system configuration...
updating GRUB 2 menu...
```



## Troubleshooting

### Error setting up resolv.conf

If you get an error about `resolv.conf` when entering the system:

```
[root@wopr:~]# nixos-enter --root /tmp/rpool
/run/current-system/sw/bin/nixos-enter: failed to set up resolv.conf
setting up /etc...
```

Temporarily replace `/etc/resolv.conf`:

```
[root@wopr:/etc]# unlink /etc/resolv.conf

[root@wopr:/etc]# echo 'nameserver 8.8.8.8' > /etc/resolv.conf
```


## References

* [ArsTechnica: A quick-start guide to OpenZFS native encryption](https://arstechnica.com/gadgets/2021/06/a-quick-start-guide-to-openzfs-native-encryption/)
