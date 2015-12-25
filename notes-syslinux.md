<!---

vim: set nu ai et ts=4 sw=4 :
-->

# Some notes about syslinux

If using encryption
---

&nbsp;&nbsp;Where the decrypted container is named **crypt**, and resides on **/dev/sda2**. Change APPEND line to read:
```bash
 APPEND root=/dev/mapper/crypt cryptdevice=/dev/sda2:crypt rw
```

If using lvm
---

&nbsp;&nbsp;Where the volume group is named **Charon**, and the root volume is named **rootVol**. Change the APPEND line to read:
```bash
 APPEND root=/dev/mapper/Charon-rootVol rw
```

If using btrfs and using a subvolume as root
---

&nbsp;&nbsp;Where the btrfs subvolume is named **ROOT**, and resides on **/dev/sda2**. Change the APPEND line to read:
```bash
 APPEND root=/dev/sda2 rw rootflags=subvol=ROOT
```

If using encryption and lvm
---

&nbsp;&nbsp;Where the decrypted container is named **lvm**, resides on **/dev/sda2**, the volume group is named **Charon**, and the root volume is named **rootVol**. Change the APPEND line to read:
```bash
 APPEND root=/dev/mapper/Charon-rootVol cryptdevice=/dev/sda2:lvm rw
```

If using encryption, lvm, and btrfs
---

&nbsp;&nbsp;Where the decrypted container is named **lvm**, resides on **/dev/sda2**, the volume group is named **Charon**, the root volume is named **rootVol**, and the btrfs subvolume is named **ROOT**. Change the APPEND line to read:
```bash
 APPEND root=/dev/mapper/Charon-rootVol cryptdevice=/dev/sda2:lvm rw rootflags=subvol=ROOT
```

Possible differences between permutations
---

&nbsp;&nbsp;Disk name **/dev/sda2** could be replaced with a partition labelling ie: **/dev/disk/by-partlabel/Root**.
&nbsp;&nbsp;An elevator for disk io could be specified, just add to the end of the APPEND line **elevator=noop** for noop elevator, could also use **deadline** instead of **noop**.
&nbsp;&nbsp;Do remember that some of these will need to be specified in your **fstab** file as well.

<!---
![wifi bars](/home/ygoreus/Downloads/wifi-icon-1.png)
-->

