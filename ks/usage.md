如何制作CentOS 7.5自动安装U盘
==========================

### 1. 下载ISO镜像
用自己熟悉的工具，从国内镜像下载速度较快，例如阿里镜像下载地址
```
https://mirrors.aliyun.com/centos/7.5.1804/isos/x86_64/CentOS-7-x86_64-Minimal-1804.iso
```
下载完成记得校验sha1sum: 13675c6f74880e7ff3481b91bdaf925ce81bda8f

### 2. 下载制作工具 [Rufus](https://rufus.akeo.ie/)
成功制作启动U盘的关键工具，其他同类工具大多已经失效
```
https://rufus.akeo.ie/downloads/rufus-2.18p.exe
```
绿色软件，下载保存，直接运行，我在项目bin目录下备份了一份，sha1sum: d0054fc533603004a107436f47bc020afd54fa05

### 3. 制作USB安装盘

插入U盘，选择ISO镜像，缺省值，无需更改，点击开始
![如图](img/rufus.png)

### 4. 配置kickstart修改密码或添加内容

缺省密码`123456`，如果需要修改，请按以下步骤操作

1. 生成root密码
```
$ python -V
Python 2.7.15
$ python -c 'import crypt, getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'
Password: 123456
$6$W8q1a2rLmDGblV9i$1CDQR2zUp4c7eGwFqWqByyxg59BVKg4tjU3gHVVYJ.E5p71VGYSaYVDR2hDP.7c5uFuvnoIgUpuHSH4UVn9y4.
```
2. 修改ks.cfg配置，用上面生成的密码替换
```
rootpw --iscrypted $6$W8q1a2rLmDGblV9i$1CDQR2zUp4c7eGwFqWqByyxg59BVKg4tjU3gHVVYJ.E5p71VGYSaYVDR2hDP.7c5uFuvnoIgUpuHSH4UVn9y4.
```
3. 校验ks.cfg错误
```
$ sudo yum install pykickstart
$ ksvalidator ks.cfg
```

### 5. 编辑启动菜单项
修改使用Rufus制作好的U盘
- ks.cfg复制isolinux/ks.cfg
- isolinux.cfg替换isolinux/isolinux.cfg
- grub.cfg替换EFI/BOOT/grub.cfg

> 以下只是说明，无需动手操作

isolinux/isolinux.cfg添加了`inst.ks=hd:LABEL=CENTOS\x207\x20X8:/isolinux/ks.cfg`
```
label linux
  menu label KS ^Install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img inst.ks=hd:LABEL=CENTOS\x207\x20X8:/isolinux/ks.cfg inst.stage2=hd:LABEL=CENTOS\x207\x20X8 quiet
```

支持EFI启动的关键是创建/boot/efi分区，文件系统类型efi，ks.cfg添加的内容：
```
part /boot/efi --fstype "efi" --size 128 --asprimary --ondisk $disk
```

EFI/BOOT/grub.cfg，添加了`inst.ks=hd:LABEL=CENTOS\x207\x20X8:/isolinux/ks.cfg`
```
menuentry 'EFI Install CentOS 7' --class fedora --class gnu-linux --class gnu --class os {
    linuxefi /images/pxeboot/vmlinuz inst.ks=hd:LABEL=CENTOS\x207\x20X8:/isolinux/ks.cfg inst.stage2=hd:LABEL=CENTOS\x207\x20X8 quiet
    initrdefi /images/pxeboot/initrd.img
}
```

>补充说明：安装盘的卷标为`CentOS 7 x86_64`，空格为十六进制\x20，所以`LABEL=CENTOS\x207\x20X8`，制作U盘时，Rufus已经自动修改好isolinux.cfg和grub.cfg

### 6. 使用VirtualBox测试安装盘
如何[下载安装](https://download.virtualbox.org/virtualbox/5.2.8/VirtualBox-5.2.8-121009-Win.exe)，操作步骤此处省略，您可以搜索相关文档，关键的两点说明如下：

1. 从物理盘创建虚拟盘
先插好U盘，运行`diskmgmt.msc`获取移动硬盘序号PysicalDrive`1`，再以**管理员身份运行**`cmd.exe`输入如下命令：
```
C:\Program Files\Oracle\VirtualBox>
VBoxManage.exe internalcommands createrawvmdk -filename C:\usb.vmdk -rawdisk \\.\PhysicalDrive1
```
把创建好的usb.vmdk添加到`设置->存储->控制器：IDE`中

2. 使用主机输入输出I/O缓存
以**管理员身份运行**VirtualBox，`设置->存储->控制器：SATA`，☑使用主机输入输出（I/O）缓存 ![如图](img/usboot.png)

### 7. 使用U盘启动安装系统
各品牌进入启动菜单选择项的快捷键不同，最常见的是**F12**
![KS Install CentOS 7](img/ksins.png)
按快捷键`i`或使用方向键选择`KS Install CentOS 7`回车，后面就无需再干预了，会自动完成安装并重启系统。

### 8. 在线扩容

ks.cfg定义的分区策略是
```
/boot/efi 128M
/boot     512M
/         20G
swap      Auto
```

登录系统`/`分区运态扩容，例如从缺省20g扩展到30g
```
# lvs /dev/lvg2/root
  LV   VG   Attr       LSize
  root lvg2 -wi-ao---- 20.00g
# lvextend -L +10g /dev/lvg2/root
  Size of logical volume lvg2/root changed from 30.00 GiB (640 extents) to 20.00 GiB (9600 extents).
  Logical volume lvg2/root successfully resized.
# resize2fs /dev/lvg2/root
redisze2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/lvg2/root is mounted on /; on-line resizing required
old_desc_blocks = 3, new_desc_lobcks = 4
The filesystem on /dev/lvg2/root is now 7864320 blocks long.
# lvs /dev/lvg2/root
  LV   VG   Attr       LSize
  root lvg2 -wi-ao---- 30.00g
```

> 补充说明xfs文件系统使用`xfs_growfs`扩容

### 9. 系统优化说明
* 时区 Asia/Shanghai
* 禁止 SELinux
* 关闭 Firewall
* 镜像 mirrors.aliyun.com
* 授时 ntp.aliyun.com
* 最大文件打开数 nofile 65535

参考：
- https://www.aioboot.com/en/boot-from-usb-in-virtualbox/
- https://www.howtogeek.com/187721/how-to-boot-from-a-usb-drive-in-virtualbox/
- https://www.pendrivelinux.com/boot-a-usb-flash-drive-in-virtualbox/
- https://github.com/DavidBrenner3/VMUB/releases
