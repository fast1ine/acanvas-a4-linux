# LG Acanvas A4 — mainline Linux

Linux 6.18 (LTS) for the LG Electronics "Acanvas A4", an i.MX6 Solo battery-powered 23" portrait signage
frame (1 GiB RAM, 8 GB eMMC, dual-channel LVDS panel, Intel 3160 Wi-Fi/BT, motorised charging puck).

- `dts/imx6dl-lge-a4.dts` — board device tree (`imx6dl.dtsi` + `imx6qdl-sabresd.dtsi` + the A4 differences)
- `patches/0001-ARM-dts-imx-add-LG-Acanvas-A4-board.patch` — the same DTS plus the Makefile entry, against Linux v6.18.54
- `config/a4.config` — Kconfig fragment merged on top of `imx_v6_v7_defconfig`

The stock U-Boot 2014.04 is used unmodified and the Debian userland is not part of this repository.

# How to build

1. Get Linux 6.18.54
   ```
   wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.54.tar.xz
   tar xf linux-6.18.54.tar.xz
   ```

2. Download this repository and apply the board patch
   ```
   git clone https://github.com/fast1ine/acanvas-a4-linux.git
   cd linux-6.18.54
   patch -p1 < ../acanvas-a4-linux/patches/0001-ARM-dts-imx-add-LG-Acanvas-A4-board.patch
   ```

3. Build (needs an `arm-linux-gnueabihf-` cross toolchain)
   ```
   export ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
   make imx_v6_v7_defconfig
   scripts/kconfig/merge_config.sh -m .config ../acanvas-a4-linux/config/a4.config
   make olddefconfig
   make -j$(nproc) zImage modules nxp/imx/imx6dl-lge-a4.dtb
   cat arch/arm/boot/zImage arch/arm/boot/dts/nxp/imx/imx6dl-lge-a4.dtb > zImage-lge-a4-appended
   make INSTALL_MOD_PATH=modules INSTALL_MOD_STRIP=1 modules_install
   ```
   The stock U-Boot runs `bootz` without an fdt argument, so the DTB is appended to the zImage
   (`CONFIG_ARM_APPENDED_DTB`). `zImage-lge-a4-appended` must stay under 16 MiB, the size of the kernel slot
   (about 6.3 MiB with the XZ kernel).

# How to flash

On the device as root (kernel slot = p6, root filesystem = p7 on the examined unit):
```
dd if=zImage-lge-a4-appended of=/dev/mmcblk3p6 bs=1M conv=fsync
cp -a modules/lib/modules/6.18.54-lge-a4 /lib/modules/ && depmod -a 6.18.54-lge-a4
```

U-Boot environment (user-area offset 0x80000, 8 KiB with a CRC32 header; set with `setenv` + `saveenv` on the
serial console, `ttymxc0` 115200, any key within 1 s of power-on). These variables boot this kernel; the rest
of the stock environment is left as is:
```
kernel_addr=0x9C000
kernel_size=0x8000
rootfs=PARTUUID=<PARTUUID of the root partition>
bootargs_base=setenv bootargs console=tty0 console=${console},${baudrate} fbcon=rotate:3
bootargs_mmc=setenv bootargs ${bootargs} root=${rootfs} rootwait
read_kernel=mmc dev ${mmc_dev}; mmc read ${loadaddr} ${kernel_addr} ${kernel_size}
bootcmd_mmc=run bootargs_base bootargs_mmc read_kernel; bootz ${loadaddr}
motor_hold=mw.l 0x020e022c 0x5; mw.l 0x020e0230 0x5; mw.l 0x0209c000 0x22000030; mw.l 0x0209c004 0x22000030
bootcmd=run motor_hold; run check_lastboot; run bootcmd_mmc
bootdelay=1
```
`kernel_addr`/`kernel_size` are the slot's start sector and length (16 MiB). The stock partitions p1–p5
(env, nvram, id, recovery kernel and recovery rootfs) are never written; `env default -a; saveenv` returns
the device to the stock firmware.

**Safety note.** The reel-motor request lines GPIO1_4 (UP) and GPIO1_5 (DOWN) are active low and must never
float: the DTS pulls the pads up (`0x1b0b0`) and `motor_hold` drives them idle during boot. Floating lines
made the reel wind or unwind by itself, and repeated stalls damaged the motor of the examined unit. Anything
driving these lines from user space must hold them as outputs at all times.

# License

GPL-2.0 (see [LICENSE](LICENSE)). The device tree is additionally available under the MIT license (SPDX header in the file).
