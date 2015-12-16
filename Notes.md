# Install Notes

### Disk Preparation

Erase the target disk **/dev/sda** and wipe out the previous partition table.
```bash
 sgdisk --zap-all /dev/sda
```

Partition the disk so as to use encryption and lvm. Using a 1GiB **/boot** partition
and **/** taking up the rest of the disk space. Note: Leaving 2MiB of disk space is 
required if using the grub bootloader on a gpt partition scheme.
```bash
 parted -a optimal /dev/sda mklabel gpt mkpart primary 2MiB 1026MiB name 1 Root \
                            mkpart primary 1026MiB 100% name 2 Root set 2 lvm on
```

Encrypt the larger partition using luks, leave the smaller as unencrypted as linux 
cannot load from an encrypted partition (at least not yet).
```bash
 cryptsetup --verbose --cipher aes-xts-plain64 --hash sha-512 --key-size 512 --use-urandom \
            --iter-time 5000 --verify-passphrase luksFormat /dev/sda
```
Now open the encrypted container and choose a mount name for it **lvm** will be the mount point.
```bash
 cryptsetup luksOpen /dev/sda lvm
```

Use LVM to make 2 volumes on the encrypted container a swapvolume and a rootvolume
    * Create a physical volume for the container **/dev/mapper/lvm**.
    ```bash
     pvcreate /dev/mapper/lvm 
    ```
    * Create a volume group, name it, and add the physical volume to it. **Charon** will be used as volume group name.
    ```bash
     vgcreate Charon /dev/mapper/lvm
    ```

Decide which filesystem to use for formatting, **btrfs** will be used here. Format 


#### Erase Disk



### 
