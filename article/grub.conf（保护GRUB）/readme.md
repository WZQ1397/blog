### 保护grub
```bash
# grub.conf generated by anaconda

#

# Note that you do not have to rerun grub after making changes to this file

# NOTICE:  You have a /boot partition.  This means that

#          all kernel and initrd paths are relative to /boot/, eg.

#          root (hd0,0)

#          kernel /vmlinuz-version ro root=/dev/mapper/VolGroup-lv_root

#          initrd /initrd-[generic-]version.img

#boot=/dev/vda

default=0                    //默认第一个操作系统

timeout=5                    //停留5秒

splashimage=(hd0,0)/grub/splash.xpm.gz            //允许出现的GRUB背景的path

hiddenmenu                //这个是可选的

title CentOS (2.6.32-358.el6.x86_64)

        root (hd0,0)

        kernel /vmlinuz-2.6.32-358.el6.x86_64 ro root=/dev/mapper/VolGroup-lv_root

 rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD rd_LVM_LV=VolGroup/lv_swap SYSFONT=latarcyrh

eb-sun16 crashkernel=auto rd_LVM_LV=VolGroup/lv_root  KEYBOARDTYPE=pc KEYTABLE=us

rd_NO_DM rhgb quiet

        initrd /initramfs-2.6.32-358.el6.x86_64.img        //初始化内核
```
目录：/boot/grub/grub.conf

1.明文密码    password=[xxxxxx]
2.密文密码    password --md5 [$1$nTHIS$tBDk5a1FCkB2l4pEDfQ4y1]
密码写在 title上，title下加上lock（可选）：加了每次启动都要输入，不加只在进入grub才需要密码