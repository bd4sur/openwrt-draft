# RA67 (Redmi AX5)

用于备份降级固件、原厂分区数据和[第三方UBoot](https://mbd.pub/o/bread/mbd-Ypqbk5dr)（自购付费软件，此处加密，仅供自用备份，解压密码为米3管理密码）。

## 文件列表

- `miwifi_ra67_firmware_63805_1.0.16.bin`：原厂降级固件，用于开启SSH。CRC32=`3A151209`
- `openwrt-ipq60xx-Redmi_AX5-squashfs-nand-factory.ubi`：过渡固件的根文件系统。CRC32=`DE26E06A`
- `AX5_UBoot_MIBIB.7z`：加密后的UBoot和大分区表。CRC32=`A08CDDA0`

## 刷机方法参考

1、系统版本降级到：`miwifi_ra67_all_f3fac_1.0.26.bin`或者`miwifi_ra67_firmware_63805_1.0.16.bin`。

2、登录原厂管理后台，地址栏中输入以下地址，其中<STOK>是当前session的token，确保每次返回的都是code:0。执行完毕后，官方固件SSH开启。

```
http://192.168.31.1/cgi-bin/luci/;stok=<STOK>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%3B%20nvram%20set%20ssh_en%3D1%3B%20nvram%20commit%3B%20sed%20-i%20's%2Fchannel%3D.*%2Fchannel%3D%5C%22debug%5C%22%2Fg'%20%2Fetc%2Finit.d%2Fdropbear%3B%20%2Fetc%2Finit.d%2Fdropbear%20start%3B

http://192.168.31.1/cgi-bin/luci/;stok=<STOK>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%3B%20echo%20-e%20'admin%5Cnadmin'%20%7C%20passwd%20root%3B
```

3、通过SSH访问后台，账号密码为root/admin。

```
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa root@192.168.31.1
```

4、设置env：

```
nvram set flag_last_success=0
nvram set flag_boot_rootfs=0
nvram set flag_boot_success=1
nvram set flag_try_sys1_failed=0
nvram set flag_try_sys2_failed=0
nvram set boot_wait=on
nvram set uart_en=1
nvram set telnet_en=1
nvram set ssh_en=1
nvram commit
```

5、执行`cat /proc/mtd`查看rootfs分区号。我的机器是mtd18。NAND闪存空间被分为多个分区，其中mtd18和mtd19这两个分区存储路由器的固件，对应分区名为“root_fs”和“rootfs_1”。

6、备份分区。执行`dd if=/dev/mtd1 of=/tmp/mtd1_0MIBIB.bin`（原厂分区表）和`dd if=/dev/mtd7 of=/tmp/mtd7_0APPSBL.bin`（原厂UBoot），通过scp工具将其备份到本地。

7、通过scp将固件的ubi镜像`openwrt-ipq60xx-Redmi_AX5-squashfs-nand-factory.ubi`传到路由器`/tmp`目录。

8、写入固件到rootfs分区（上面已经看到rootfs是`/dev/mtd18`），并重启。这个固件就是刷扩容固件时用到的所谓“过渡固件”，以它为跳板，刷入扩容版固件，结合第三方分区表和UBoot，即可从扩容版固件启动，同时获得融合后的大分区。

```
# 将固件镜像写入rootfs分区，如果报错，则改成rootfs_1所在分区。
ubiformat /dev/mtd18 -y -f /tmp/openwrt-ipq60xx-Redmi_AX5-squashfs-nand-factory.ubi
```

9、切换下一轮启动的分区：

```
# 注意：下面两个flag，如果刷入的是rootfs，则为0；如果刷入的是rootfs_1，则为1。
nvram set flag_last_success=0
nvram set flag_boot_rootfs=0
nvram commit
reboot
```

10、断电重启路由器，通过静态IP=192.168.1.1访问路由器后台，用户名密码为root/password或者空。

11、通过SCP将UBoot和大分区表上传到`/tmp`目录，再将其刷入指定分区：

```
cd /tmp
mtd write mibib.bin /dev/mtd1
mtd write uboot.bin /dev/mtd7
```

12、路由器断电，然后按住reset按键开机，黄灯闪5下后变蓝，一直按住reset键，浏览器通过静态IP=192.168.1.1打开固件更新界面后松开reset键，即进入第三方UBoot，可刷入factory固件。
