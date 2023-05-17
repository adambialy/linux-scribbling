## 1. install shizzle

```
apt-get install multipath-tools open-iscsi -y
```

## 2. run discovery

```
# iscsiadm -m discovery -t st -p 192.168.14.100
192.168.14.110:3260,1 iqn.1988-11.com.dell:01.array.bc305bf24f7d
192.168.14.112:3260,2 iqn.1988-11.com.dell:01.array.bc305bf24f7d
192.168.14.111:3260,3 iqn.1988-11.com.dell:01.array.bc305bf24f7d
192.168.14.113:3260,4 iqn.1988-11.com.dell:01.array.bc305bf24f7d
192.168.14.100:3260,5 iqn.1988-11.com.dell:01.array.bc305bf24f7d
192.168.14.102:3260,6 iqn.1988-11.com.dell:01.array.bc305bf24f7d
192.168.14.101:3260,7 iqn.1988-11.com.dell:01.array.bc305bf24f7d
192.168.14.103:3260,8 iqn.1988-11.com.dell:01.array.bc305bf24f7d
```

## 3. tweak multipath.conf

multipath.conf

```
defaults {
    udev_dir        /dev
    polling_interval 10
    path_grouping_policy    multibus
    getuid_callout  "/lib/udev/scsi_id --whitelisted --device=/dev/%n"
    prio_callout    /sbin/mpath_prio_alua /dev/%n
    path_checker    directio
    rr_min_io       100
    failback        immediate
    no_path_retry   5
    user_friendly_names yes
}

blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^hd[a-z]"
    devnode "^cciss.*"
}

devices {
    device {
        vendor                  "DELL"
        product                 "ME4"
        path_grouping_policy    multibus
        path_checker            tur
        prio_callout            "/sbin/mpath_prio_alua /dev/%n"
        hardware_handler        "0"
        failback                15
        rr_min_io               100
        rr_weight               uniform
        no_path_retry           18
        features                "0"
    }
}
```


section for dell

```
# dell-me4
    device {
        vendor "DellEMC"
        product "ME4"
        path_grouping_policy "group_by_prio"
        path_checker "tur"
        hardware_handler "1 alua"
        prio "alua"
        failback followover
        rr_weight "uniform"
        path_selector "round-robin 0"
        max_sectors_kb 8192
    }
```
 
## 4. accept request from `grep -v "#" /etc/iscsi/initiatorname.iscsi

```
InitiatorName=iqn.2004-10.com.ubuntu:01:4f598a60af82
```

## 5. enable things

```
systemctl restart open-iscsi.service 
systemctl enable open-iscsi.service 
systemctl enable multipathd
systemctl enable multipath-tools
systemctl restart multipathd
systemctl restart multipath-tools.service
```

## 6. iscsid.conf:

```
node.startup = automatic
node.leading_login = No
```


## 7. configure interface

```
iscsiadm -m iface -I ens192 --op=new
iscsiadm -m iface -I ens192 --op=update -n iface.hwaddress -v 00:50:56:b4:9f:36
iscsiadm -m iface -I ens192 --op=update -n iface.ipaddress -v 192.168.14.106
```

figure out uuid with blkid, and add to fstab

```
# blkid 
/dev/sda2: UUID="8868e844-7568-44c0-9f07-17b19e571541" TYPE="ext4" PARTUUID="76cdf050-c7e4-4619-9eda-7d141518e6ce"
/dev/sda3: UUID="mpaVHF-AGY2-2QNv-Nsfd-3CuL-bdQ2-o04Jj1" TYPE="LVM2_member" PARTUUID="f2ab080a-ccc0-4e28-abeb-a49cafa00f94"
/dev/mapper/ubuntu--vg-ubuntu--lv: UUID="c43eae72-40cd-4cef-a9e9-9815f74d0b0e" TYPE="ext4"
/dev/loop0: TYPE="squashfs"
/dev/loop1: TYPE="squashfs"
/dev/loop2: TYPE="squashfs"
/dev/loop3: TYPE="squashfs"
/dev/loop4: TYPE="squashfs"
/dev/loop5: TYPE="squashfs"
/dev/loop6: TYPE="squashfs"
/dev/sda1: PARTUUID="15f885aa-57f5-4d0e-a9f4-9e809d142983"
/dev/sdb1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/sde1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/sdd1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/sdc1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/sdf1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/sdi1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/sdh1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/sdg1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/mapper/mpatha-part1: UUID="c23abb69-d6b9-4bc7-885f-88a637893b68" TYPE="ext4" PARTUUID="bbb30f0e-01"
/dev/mapper/mpatha: PTUUID="bbb30f0e" PTTYPE="dos"
```


/etc/fstab:

```
/dev/disk/by-uuid/c23abb69-d6b9-4bc7-885f-88a637893b68 /mnt/iscsi ext4 defaults 0 2
```

## 8. systemctl automount


/etc/systemd/system/mnt-iscsi.mount:

```
[Unit]
Description=iscsi
After=connect-luns.service
DefaultDependencies=no

[Mount]
What=/dev/disk/by-uuid/c23abb69-d6b9-4bc7-885f-88a637893b68
Where=/mnt/iscsi
Type=ext4
StandardOutput=journal
TimeoutSec=30


[Install]
WantedBy=multi-user.target
```

/etc/systemd/system/mnt-iscsi.automount:

```
[Unit]
Description=iscsit
Requires=network-online.target
#After=

[Automount]
Where=/mnt/iscsi

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload
systemctl enable mnt-iscsi.automount 
systemctl enable mnt-iscsi.mount 
```

## Links

https://github.com/f1linux/iscsi-automount









