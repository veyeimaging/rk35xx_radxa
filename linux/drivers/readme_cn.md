# 怎样将VEYE相机的驱动移植到radxa的Linux系统

驱动移植基于radxa的bsp构建环境，硬件平台Radxa ZERO 3W，linux内核版本5.10。

## 准备工作

在驱动移植之前，确保已经编译过一次Radxa标准版本内核，命令如下：

```
cd bsp
./bsp linux rk356x
```

首次编译时，会从kernel仓库下载最新代码，源码的路径位于 bsp 目录下的 .src/linux。

## 移植流程

1. 把cam_drv_src目录下的源码复制到内核目录：.src/linux/drivers/media/i2c。

2. 修改.src/linux/drivers/media/i2c/Kconfig，增加内容：

```bash
config VIDEO_DS90UB954
       tristate "TI FDPLINK III support."
       depends on I2C && VIDEO_V4L2 && VIDEO_V4L2_SUBDEV_API
       help
            Driver for TI FDPLINK III .

config VIDEO_VEYECAM2M
       tristate "VEYE 2M camera support."
       depends on I2C && VIDEO_V4L2 && VIDEO_V4L2_SUBDEV_API
       depends on MEDIA_CAMERA_SUPPORT
       help
               Driver for veye 2m camera.
               
config VIDEO_VEYEMVCAM
       tristate "VEYE MV series camera support."
       depends on I2C && VIDEO_V4L2 && VIDEO_V4L2_SUBDEV_API
       depends on MEDIA_CAMERA_SUPPORT
       help
               Driver for veye mv series camera.
```

3. 修改.src/linux/drivers/media/i2c/Kconfig/Makefile，增加内容：

```bash
obj-$(CONFIG_VIDEO_DS90UB954) += ds90ub954.o
obj-$(CONFIG_VIDEO_VEYECAM2M) += veyecam2m.o
obj-$(CONFIG_VIDEO_VEYEMVCAM) += veye_mvcam.o
```

4. 把dts目录下的设备树文件（rk3566-radxa-zero3-veyecam2m.dtsi、rk3566-radxa-zero3-veyemvcam.dtsi和rk3566-radxa-zero-3w-aic8800ds2.dts）复制到内核目录：.src/linux/arch/arm64/boot/dts/rockchip。

5. 修改.src/linux/arch/arm64/boot/dts/rockchip/rk3566-radxa-zero-3w-aic8800ds2.dts，增加内容：

```bash
//#include "rk3566-radxa-zero3-veyecam2m.dtsi"
#include "rk3566-radxa-zero3-veyemvcam.dtsi"
```

根据相机类型，以上内容二选一。其中veyecam2m对应VEYE系列彩色相机。veyemvcam对应MV系列和RAW系列相机。

6. 修改bsp/linux/rk356x/kconfig.conf, 增加内容：

```
# CONFIG_VIDEO_VEYECAM2M=y
CONFIG_VIDEO_VEYEMVCAM=y
```

根据相机类型，以上内容二选一。

7. 编译内核deb包

先查看当前linux版本，确认用户版本号。如下信息中，用户版本号为26。

```
uradxa@radxa-zero3:~$ uname -a
Linux radxa-zero3 5.10.160-26-rk356x #27 SMP Tue Apr 16 13:47:32 UTC 2024 aarch64 GNU/Linux
```

编译内核：

```
./bsp --no-prepare-source linux rk356x -r 36

参数说明：
--no-prepare-source  # 使用本地修改进行编译，如果不加这个参数将会从 Radxa kernel 仓库同步最新代码并覆盖本地修改。
-r 36                # 由于板子当前的用户版本号为26，所以指定新的版本号必须大于等于26。

```
注意：在使用-r时，版本号必须大于等于板子中当前的用户版本号。
编译完成后，把bsp目录下以下安装包复制到板子上。

```
linux-headers-5.10.160-36-rk356x_5.10.160-36_arm64.deb
linux-image-5.10.160-36-rk356x_5.10.160-36_arm64.deb
```

8. 安装deb包

```
sudo dpkg -i linux-headers-5.10.160-36-rk356x_5.10.160-36_arm64_cameratype.deb
sudo dpkg -i linux-image-5.10.160-36-rk356x_5.10.160-36_arm64_cameratype.deb
sudo reboot
```

使用uname -a查看是否安装成功

## bsp编译常见问题
### 1、docker权限问题
警告信息如下：

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/images/json?filters=%7B%22reference%22%3A%7B%22ghcr.io%2Fradxa-repo%2Fbsp%3Amain%22%3Atrue%7D%7D": dial unix /var/run/docker.sock: connect: permission denied
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/images/create?fromImage=ghcr.io%2Fradxa-repo%2Fbsp&tag=main": dial unix /var/run/docker.sock: connect: permission denied
```

解决方案:
```
sudo chmod 666 /var/run/docker.sock
```

## 2、网络问题

执行如下命令时，

```
./bsp --no-prepare-source linux rk356x -r 21
```

虽然bsp不会从代码仓库下载源码，但是要求linux系统能够访问互联网，否则编译失败。提示信息如下：

```
rror response from daemon: Get "https://ghcr.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

## 其他
### 查看板子使用的设备树

在编译内核安装包时，编译了多种板子的设备树，可通过如下方式确实板子实际使用的设备树。

1. 在板子断电状态下，连接计算机和板子的串口，然后板子加电。
2. 按下ctrl+c，进入uboot命令行模式。
3. 输入printenv ,确认fdtfile变量的值。

```
=> printenv 
fdtfile=rockchip/rk3566-radxa-zero-3w-aic8800ds2.dtb
```

从上述fdtfile的值可确定, 内核中对应的DTS文件为`rk3566-radxa-zero-3w-aic8800ds2.dts`。


## 参考资料
[Radxa Zero 3W docs](https://docs.radxa.com/en/zero/zero3)

[BSP toolkit](https://radxa-repo.github.io/bsp/)
