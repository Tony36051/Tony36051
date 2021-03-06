---
title: T1刷机Armbian经验
date: 2021-03-03

---


T1同时保留安卓和Armbian两个系统。

<!--more-->

# 刷机
## 刷安卓
开晨星的刷机工具（USB_Burning_Tool），加载固件，USB双公头，root后在安卓的`终端模拟器`中用reboot update命令重启。
检测到机器后，开刷，等待完成就可以拔线进入安卓系统。
## 刷Armbian
使用老的内核镜像3.14，配套`gxm_q201_2g.dtb`，用`usb_image_tool`刷到的U盘后，修改BOOT盘，将`gxm_q201_2g.dtb`移动到BOOT的根目录改名为`dtb.img`。
我用的镜像名是`Armbian_5.44_S9xxx_Ubuntu_xenial_3.14.29_server`
插入U盘，重启盒子，进入Armbian。默认账号root，默认密码是1234.
修改密码后，Ctrl+C跳过创建用户。
同时保留两个系统的关键是进入Armbian后，修改/root/install.sh文件为以下内容，内容转自恩山论坛大佬achaoge。
这里文件适用于首次安装Armbian，非首次安装请查看恩山上他的原贴。
```bash
#!/bin/sh

echo "Start copy system for DATA partition."

mkdir -p /ddbr
chmod 777 /ddbr

VER=`uname -r`

IMAGE_KERNEL="/boot/zImage"
IMAGE_INITRD="/boot/initrd.img-$VER"
PART_ROOT="/dev/data"
DIR_INSTALL="/ddbr/install"
IMAGE_DTB="/boot/dtb.img"


if [ ! -f $IMAGE_KERNEL ] ; then
    echo "Not KERNEL.  STOP install !!!"
    return
fi

if [ ! -f $IMAGE_INITRD ] ; then
    echo "Not INITRD.  STOP install !!!"
    return
fi

#edit by achaoge,
#disable 64bit and metadata_csum features for uboot compatibility
#ref: https://kshadeslayer.wordpress.com/2016/04/11/my-filesystem-has-too-many-bits/
/sbin/resize2fs -s $PART_ROOT
/sbin/tune2fs -O ^metadata_csum $PART_ROOT

#echo "Formatting DATA partition..."
#umount -f $PART_ROOT
#mke2fs -F -q -t ext4 -m 0 $PART_ROOT
e2fsck -f $PART_ROOT
#echo "done."

echo "Copying ROOTFS."

if [ -d $DIR_INSTALL ] ; then
    rm -rf $DIR_INSTALL
fi

mkdir -p $DIR_INSTALL
mount -o rw $PART_ROOT $DIR_INSTALL

cd /
echo "Copy BIN"
tar -cf - bin | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy BOOT"
#mkdir -p $DIR_INSTALL/boot
tar -cf - boot | (cd $DIR_INSTALL; tar -xpf -)
echo "Create DEV"
mkdir -p $DIR_INSTALL/dev
#tar -cf - dev | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy ETC"
tar -cf - etc | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy HOME"
tar -cf - home | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy LIB"
tar -cf - lib | (cd $DIR_INSTALL; tar -xpf -)
echo "Create MEDIA"
mkdir -p $DIR_INSTALL/media
#tar -cf - media | (cd $DIR_INSTALL; tar -xpf -)
echo "Create MNT"
mkdir -p $DIR_INSTALL/mnt
#tar -cf - mnt | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy OPT"
tar -cf - opt | (cd $DIR_INSTALL; tar -xpf -)
echo "Create PROC"
mkdir -p $DIR_INSTALL/proc
echo "Copy ROOT"
tar -cf - root | (cd $DIR_INSTALL; tar -xpf -)
echo "Create RUN"
mkdir -p $DIR_INSTALL/run
echo "Copy SBIN"
tar -cf - sbin | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy SELINUX"
tar -cf - selinux | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy SRV"
tar -cf - srv | (cd $DIR_INSTALL; tar -xpf -)
echo "Create SYS"
mkdir -p $DIR_INSTALL/sys
echo "Create TMP"
mkdir -p $DIR_INSTALL/tmp
echo "Copy USR"
tar -cf - usr | (cd $DIR_INSTALL; tar -xpf -)
echo "Copy VAR"
tar -cf - var | (cd $DIR_INSTALL; tar -xpf -)

echo "Copy fstab"

rm $DIR_INSTALL/etc/fstab
cp -a /root/fstab $DIR_INSTALL/etc
#cp -a /boot/hdmi.sh $DIR_INSTALL/boot

#add by achaoge 2018-06-22
export $(/usr/sbin/fw_printenv mac)
echo "Modify files for N1 emmc boot"
/bin/sed -e "/usb [23]/d" -e 's/fatload mmc 0 \([^ ]*\) \([^;]*\)/ext4load mmc 1:c \1 \/boot\/\2/g' -i $DIR_INSTALL/boot/s905_autoscript.cmd
/bin/sed -e 's/LABEL=ROOTFS/\/dev\/data/' -e "s/mac=.*/mac=${mac}/" -i $DIR_INSTALL/boot/uEnv.ini
/usr/bin/mkimage -C none -A arm -T script -d $DIR_INSTALL/boot/s905_autoscript.cmd $DIR_INSTALL/boot/s905_autoscript
echo "Emmc boot fixed end"

rm $DIR_INSTALL/root/install.sh
rm $DIR_INSTALL/root/fstab
rm $DIR_INSTALL/usr/bin/ddbr
rm $DIR_INSTALL/usr/bin/ddbr_backup_nand
rm $DIR_INSTALL/usr/bin/ddbr_restore_nand

cd /
sync

umount $DIR_INSTALL

echo "*******************************************"
echo "Done copy ROOTFS"
echo "*******************************************"

#echo "Writing new kernel image..."

#mkdir -p $DIR_INSTALL/aboot
#cd $DIR_INSTALL/aboot
#dd if=/dev/boot of=boot.backup.img
#abootimg -i /dev/boot > aboot.txt
#abootimg -x /dev/boot
#abootimg -u /dev/boot -k $IMAGE_KERNEL
#abootimg -u /dev/boot -r $IMAGE_INITRD
#
#echo "done."

#if [ -f $IMAGE_DTB ] ; then
#    abootimg -u /dev/boot -s $IMAGE_DTB
#    echo "Writing new dtb ..."
#    dd if="$IMAGE_DTB" of="/dev/dtb" bs=262144 status=none && sync
#    echo "done."
#fi

echo "Write env bootargs"
#/usr/sbin/fw_setenv initargs "root=/dev/data rootflags=data=writeback rw console=ttyS0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.repair=yes net.ifnames=0 mac=\${mac}"

#Edit by achaoge 2018-06-22, for Phicomm N1 boot from emmc
/usr/sbin/fw_setenv start_autoscript "if usb start ; then run start_usb_autoscript; fi; if ext4load mmc 1:c 1020000 /boot/s905_autoscript; then autoscr 1020000; fi;"


echo "*******************************************"
echo "Complete copy OS to eMMC parted DATA"
echo "*******************************************"

```

# 两边切换
## Armbian切换安卓
```bash
mv /boot/s905_autoscript /boot/s905_autoscript.bak
```

## 安卓切换Armbian
启动安卓的`终端模拟器`中，先su切换root用户，然后执行命令
```bash
su
mv /data/boot/s905_autoscript.bak /data/boot/s905_autoscript 
```