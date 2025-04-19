# RE-CS-02 (京东云雅典娜AX6600)

用于备份[第三方UBoot](https://mbd.pub/o/bread/mbd-ZpeWlZts)（自购付费软件，此处加密，仅供自用备份，解压密码为米3管理密码）。

## 文件列表

- `JDC_AX6600_uboot_gpt.7z`：加密后的UBoot和分区表。CRC32=`529F3E53`
- `mmcblk0p10_0CDT_RE-CS-02_xxx.bin`：分别是1GB、2GB、4GB内存的CDT分区文件。其中4GB的实测可用，由于IPQ60xx的限制，最多可识别3GB。带Original的是从原厂CDT中dump出来的。4GB版的CRC32=`4A3BAA0C`

## 刷UBoot方法参考(TTL，1GB内存+原厂固件r2204已验证)

[更多教程参考此处](https://github.com/lgs2007m/Actions-OpenWrt/releases)

1. 电脑网口设置为静态IP=192.168.10.1，连接到雅典娜LAN口。拆开前面板，连接调试串口，不接Vcc，电平3V3。

2. 电脑上打开TFTPD64，设置根目录为UBoot的`uboot.bin`和`gpt.bin`文件所在目录，设置IP=192.168.10.1。

3. Putty连接串口，波特率115200，按住reset，上电，不停按回车进入6018#。

4. 刷uboot.bin

```
tftpboot uboot.bin && flash 0:APPSBL && flash 0:APPSBL_1
```

5. 刷gpt.bin分区（可以用原厂 GPT 分区, 也可用亚瑟2048M的gpt）

```
tftpboot gpt.bin && mmc erase 0x0 0x21 && mmc write 0x44000000 0x0 0x21
```

6. 清空bootconfig

```
mmc erase 0x622 0x200
mmc erase 0x822 0x200
```

7. 路由器断电，然后按住reset按键开机，黄灯闪5下后变蓝，一直按住reset键，浏览器通过静态IP=192.168.1.1打开固件更新界面后松开reset键，即进入第三方UBoot，可刷入factory固件。
