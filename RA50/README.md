# RA50 (Redmi AX5-JDC)

- 用于备份降级固件、原厂分区数据和[第三方UBoot](https://mbd.pub/o/bread/mbd-YpqUlJhy)（自购付费软件，此处加密，仅供自用备份，解压密码为米3管理密码）。
- 分为1GB内存版和512MB内存版，1GB内存版需要刷写CDT分区及其冗余分区。

## 文件列表

- `miwifi_ra50_firmware_a69aa_1.1.47.bin`：原厂降级固件，用于开启SSH。CRC32=`E16C08B8`
- `kernel.bin`：过渡固件的内核文件。CRC32=`46BA2867`
- `rootfs.7z`：过渡固件的根文件系统，由于文件过大，分成两个压缩分卷。最终解压出的`rootfs.bin`其CRC32=`F0137A65`
- `AX5_JDC_UBoot_MIBIB.7z`：加密后的UBoot和大分区表。CRC32=`EFF5544B`
- `CDT_IPQ6000_1G.bin`：CDT分区数据（两个CDT分区冗余保存同一份数据），用于1GB大内存支持，注意事项见后文。CRC32=`04F3CD35`

## 刷机方法参考

1、系统版本降级到原厂1.1.47。

2、登录原厂管理后台，地址栏中输入以下地址，确保每次返回的都是code:0。执行完毕后，官方固件SSH开启。

```
http://192.168.31.1/cgi-bin/luci/;stok=<TOKEN>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%0anvram%20set%20ssh_en=1%0anvram%20commit%0a

http://192.168.31.1/cgi-bin/luci/;stok=<TOKEN>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%0aecho%20sed%20-i%20's/channel=.*/channel="debug"/g'%20/etc/init.d/dropbear%20>%20/tmp/r.sh%0ash%20/tmp/r.sh%0arm%20/tmp/r.sh%0a

http://192.168.31.1/cgi-bin/luci/;stok=<TOKEN>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%0a/etc/init.d/dropbear%20start%0a

http://192.168.31.1/cgi-bin/luci/;stok=<TOKEN>E/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%0a/etc/init.d/dropbear%20enable%0a

http://192.168.31.1/cgi-bin/luci/;stok=<TOKEN>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%0aecho%20echo%20-e%20"root\nroot"%20|%20passwd%20root%20>%20/tmp/pass.sh%0ash%20/tmp/pass.sh%0arm%20/tmp/pass.sh%0a
```

3、通过SSH访问后台，账号密码均为root。

```
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa root@192.168.31.1
```

4、设置env：

```
nvram set flag_last_success=0
nvram set flag_boot_rootfs=0
nvram set boot_wait=on
nvram set uart_en=1
nvram set telnet_en=1
nvram set ssh_en=1
nvram set bootdelay=2
nvram commit
```

5、通过SCP将内核`kernel.bin`以及过渡OpenWrt固件`rootfs.bin`上传到路由器`/tmp`目录，并将其刷入相应分区（注意：刷写前确认分区对应设备的路径是否与下面的路径一致）：

```
dd if=/tmp/kernel.bin of=/dev/mmcblk0p17
dd if=/tmp/rootfs.bin of=/dev/mmcblk0p20
```

6、切换下一轮启动的分区：

```
nvram set flag_last_success=1
nvram set flag_boot_rootfs=1
nvram commit
```

7、断电重启路由器，通过静态IP=192.168.1.1访问路由器后台，用户名密码为root/password或者空。

8、通过SCP将UBoot和大分区表上传到`/tmp`目录，再将其刷入指定分区：

```
cd /tmp
dd if=uboot.bin of=/dev/mmcblk0p13
dd if=mibib.bin of=/dev/mmcblk0 bs=512 count=34
```

9、继续输入下面的命令后等10s重启

```
fw_setenv fsbootargs
fw_setenv bootargs
fw_setenv bootcmd bootipq
```

10、路由器断电，然后按住reset按键开机，黄灯闪5下后变蓝，一直按住reset键，浏览器通过静态IP=192.168.1.1打开固件更新界面后松开reset键，即进入第三方UBoot，可刷入factory固件。

11、内存改1GB的还需要刷CDT分区。注意：此处备份的CDT数据，是从镁光D9STQ内存+已刷第三方UBoot的机器中提取出来，不保证适用于其他机器。

```
dd if=/tmp/CDT_IPQ6000_1G.bin of=/dev/mmcblk0p10
dd if=/tmp/CDT_IPQ6000_1G.bin of=/dev/mmcblk0p11
```
