# How to Port VEYE Camera Driver to Radxa Linux System

The driver porting is based on the Radxa BSP build environment, with the hardware platform being Radxa ZERO 3W and Linux kernel version 5.10.

## Preparation

Before porting the driver, make sure to compile the Radxa standard kernel at least once using the following command:

```bash
cd bsp
./bsp linux rk356x
```

During the initial compilation, the latest code will be downloaded from the kernel repository, and the source code path will be located in the `.src/linux` directory under the `bsp` directory.

## Porting Process

1. Copy the source code from the `cam_drv_src` directory to the kernel directory: `.src/linux/drivers/media/i2c`.

2. Modify `.src/linux/drivers/media/i2c/Kconfig` and add the following content:

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
               Driver for VEYE 2M camera.
               
config VIDEO_VEYEMVCAM
       tristate "VEYE MV series camera support."
       depends on I2C && VIDEO_V4L2 && VIDEO_V4L2_SUBDEV_API
       depends on MEDIA_CAMERA_SUPPORT
       help
               Driver for VEYE MV series camera.
```

3. Modify `.src/linux/drivers/media/i2c/Kconfig/Makefile` and add the following content:

```bash
obj-$(CONFIG_VIDEO_DS90UB954) += ds90ub954.o
obj-$(CONFIG_VIDEO_VEYECAM2M) += veyecam2m.o
obj-$(CONFIG_VIDEO_VEYEMVCAM) += veye_mvcam.o
```

4. Copy the device tree files (rk3566-radxa-zero3-veyecam2m.dtsi, rk3566-radxa-zero3-veyemvcam.dtsi, and rk3566-radxa-zero-3w-aic8800ds2.dts) from the `dts` directory to the kernel directory: `.src/linux/arch/arm64/boot/dts/rockchip`.

5. Modify `.src/linux/arch/arm64/boot/dts/rockchip/rk3566-radxa-zero-3w-aic8800ds2.dts` and add the following content:

```bash
//#include "rk3566-radxa-zero3-veyecam2m.dtsi"
#include "rk3566-radxa-zero3-veyemvcam.dtsi"
```

Choose one of the above content according to the camera type. veyecam2m corresponds to VEYE series color cameras. veyemvcam corresponds to MV series and RAW series cameras.

6. Modify `bsp/linux/rk356x/kconfig.conf` and add the following content:

```
# CONFIG_VIDEO_VEYECAM2M=y
CONFIG_VIDEO_VEYEMVCAM=y
```

Choose one of the above content according to the camera type.

7. Compile the kernel deb package

First, check the current Linux version to confirm the user version number. In the following information, the user version number is 26.

```
uradxa@radxa-zero3:~$ uname -a
Linux radxa-zero3 5.10.160-26-rk356x #27 SMP Tue Apr 16 13:47:32 UTC 2024 aarch64 GNU/Linux
```

Compile the kernel:

```
./bsp --no-prepare-source linux rk356x -r 36

Parameter explanation:
--no-prepare-source  # Compile using local modifications. If this parameter is not added, it will sync the latest code from the Radxa kernel repository and override the local modifications.
-r 36                # Since the current user version number on the board is 26, specify a new version number greater than or equal to 26.
```

Note: When using `-r`, the version number must be greater than or equal to the current user version number on the board.
After compilation, copy the following installation packages from the `bsp` directory to the board.

```
linux-headers-5.10.160-36-rk356x_5.10.160-36_arm64.deb
linux-image-5.10.160-36-rk356x_5.10.160-36_arm64.deb
```

8. Install the deb packages

```
sudo dpkg -i linux-headers-5.10.160-36-rk356x_5.10.160-36_arm64_cameratype.deb
sudo dpkg -i linux-image-5.10.160-36-rk356x_5.10.160-36_arm64_cameratype.deb
sudo reboot
```

Use `uname -a` to check if the installation was successful.

## Common Issues with BSP Compilation
### 1. Docker Permission Issue

Warning messages like below may occur:

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/images/json?filters=%7B%22reference%22%3A%7B%22ghcr.io%2Fradxa-repo%2Fbsp%3Amain%22%3Atrue%7D%7D": dial unix /var/run/docker.sock: connect: permission denied
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/images/create?fromImage=ghcr.io%2Fradxa-repo%2Fbsp&tag=main": dial unix /var/run/docker.sock: connect: permission denied
```

Solution:
```
sudo chmod 666 /var/run/docker.sock
```

## 2. Network Issue

When executing the following command:

```
./bsp --no-prepare-source linux rk356x -r 21
```

Although BSP does not download source code from the repository, it requires the Linux system to access the Internet, otherwise, the compilation will fail. Prompt messages may appear as follows:

```
Error response from daemon: Get "https://ghcr.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

## Others
### Viewing the Board's Device Tree

During the compilation of kernel installation packages, multiple board device trees are compiled, and the actual device tree used by the board can be determined as follows.

1. When the board is powered off, connect the computer and the board via serial port, and then power on the board.
2. Press ctrl+c to enter the uboot command line mode.
3. Enter `printenv` to confirm the value of the `fdtfile` variable.

```
=> printenv 
fdtfile=rockchip/rk3566-radxa-zero-3w-aic8800ds2.dtb
```

From the above `fdtfile` value, it can be determined that the corresponding DTS file in the kernel is `rk3566-radxa-zero-3w-aic8800ds2.dts`.


## Reference 
[Radxa Zero 3W docs](https://docs.radxa.com/en/zero/zero3)

[BSP toolkit](https://radxa-repo.github.io/bsp/)
