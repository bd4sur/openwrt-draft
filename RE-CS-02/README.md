# RE-CS-02 (京东云雅典娜AX6600)

用于备份[第三方UBoot](https://mbd.pub/o/bread/mbd-ZpeWlZts)（自购付费软件，此处加密，仅供自用备份，解压密码为米3管理密码）。

## 文件列表

- `JDC_AX6600_uboot_gpt.7z`：加密后的UBoot和分区表。CRC32=`529F3E53`
- `mmcblk0p10_0CDT_RE-CS-02_xxx.bin`：分别是1GB、2GB、4GB内存的CDT分区文件。其中4GB的实测可用（刷写方法见下文），由于IPQ60xx的限制，最多可识别3GB。带Original的是从原厂CDT中dump出来的。4GB版的CRC32=`4A3BAA0C`
- `ipq6010-re-cs-02.dts`：将自带LED点阵小屏幕的接口改成I2C的设备树文件。编译固件前直接替换`target/linux/qualcommax/files/arch/arm64/boot/dts/qcom/ipq6010-re-cs-02.dts`。
- `re-cs-02-20250510-wifi-i2c-1024.config`：固件编译配置文件，基于`multi-20250418-base-wifi-1024.config`。具备I2C相关组件、docker相关组件和WiFi驱动。

## I2C接口定义

在刷了I2C固件的雅典娜上部署电子鹦鹉：[B站视频](https://www.bilibili.com/video/BV1mhVzzrEJf)

有三角标记的是1号脚：

|引脚编号|GPIO|功能|备注|
|----------|------|------|----|
|1|-|GND||
|2|74-586|未定义||
|3|73-585|未定义||
|4|70-582|BLSP1_I2C_SDA|3v3电平|
|5|69-581|BLSP1_I2C_SCL|3v3电平|
|6|-|3V3||
|7|-|GND||

在20250425编译的固件中，2和3引脚用作BLSP3_I2C的两个信号线，但是编译出的固件无法使用WiFi。在20250510固件中，将2和3引脚的I2C功能去掉，仅保留4和5引脚用作BLSP1_I2C，编译出的固件可以正常使用WiFi。

## 刷UBoot方法参考(TTL，1GB内存+原厂固件r2204已验证)

[更多教程参考此处](https://github.com/lgs2007m/Actions-OpenWrt/releases)

1. 电脑网口设置为静态IP=192.168.10.1，连接到雅典娜LAN口。拆开前面板，连接调试串口，不接Vcc，电平3V3。

2. 电脑上打开TFTPD64，设置根目录为UBoot的`uboot.bin`和`gpt.bin`文件所在目录，设置IP=192.168.10.1。

3. Putty连接串口，波特率115200，按住reset，上电，不停按回车进入6018#。

4. 刷uboot.bin

```
tftpboot uboot.bin && flash 0:APPSBL && flash 0:APPSBL_1
```

5. 刷gpt.bin分区（可以用原厂GPT分区, 也可用亚瑟2048M的gpt）

```
tftpboot gpt.bin && mmc erase 0x0 0x21 && mmc write 0x44000000 0x0 0x21
```

6. 清空bootconfig

```
mmc erase 0x622 0x200
mmc erase 0x822 0x200
```

7. 路由器断电，然后按住reset按键开机，黄灯闪5下后变蓝，一直按住reset键，浏览器通过静态IP=192.168.1.1打开固件更新界面后松开reset键，即进入第三方UBoot，可刷入factory固件。

## 刷CDT方法

首先刷OP固件。将本目录下的CDT文件上传到`/tmp/cdt.bin`，然后写入两个CDT分区：

```
cd /tmp
dd if=cdt.bin of=/dev/mmcblk0p10 bs=512 count=34 conv=fsync
dd if=cdt.bin of=/dev/mmcblk0p11 bs=512 count=34 conv=fsync
```

刷完CDT后重启，free可以看到内存容量数值发生变化。此时可以硬件替换内存芯片。替换后可以用memtester测试。
