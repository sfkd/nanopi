# Android 12 on NanoPi R6S with USB bluetooth adapter
## Build envrionment
* OS
  * Ubuntu 22.04
* Storage
  * 500GB
* Memory
  * 32GB

## USB bluetooth adapter
* I-O DATA USB-BT40LE
* I-O DATA USB-BT50LE
* Planex BT-Micro4
* Buffalo BSBT5D200

## Install build tools
```
$ sudo apt-get purge openjdk-* icedtea-* icedtea6-*
$ sudo apt-get update
$ sudo apt-get install openjdk-11-jdk
$ java -version
$ sudo dpkg --add-architecture i386
$ sudo apt-get update
$ sudo apt-get install bison g++-multilib git gperf libxml2-utils make zlib1g-dev:i386 zip liblz4-tool libncurses5 libssl-dev bc flex curl python-is-python3
$ sudo apt-get install device-tree-compiler python2 clang lld rsync
$ sudo ln -s /usr/bin/llvm-objdump-14 /usr/bin/llvm-objdump
$ sudo ln -s /usr/bin/llvm-ar-14 /usr/bin/llvm-ar
```

## Fetch and extract Android source
* https://dl.friendlyelec.com/nanopir6s
* Google Drive or OneDrive
* RK3588-FriendlyElec -> 07_Source codes -> rk35xx-android12-xxxxxxx-YYYYMMDD.tgz
* Extract source
```
$ tar zxvf rk35xx-android12-xxxxxxx-YYYYMMDD.tgz
```

## Modify kernel source
```
$ cd rk35xx-android12-xxxxxxx-YYYYMMDD
$ vi kernel-5.10/arch/arm64/kernel/vdso/gen_vdso_offsets.sh
< 's/^\([0-9a-fA-F]*\) . VDSO_\([a-zA-Z0-9_]*\)$/\#define vdso_offset_\2\t0x\1/p'
--
> 's/^\([0-9a-fA-F]*\) . VDSO_\([a-zA-Z0-9_]*\)$/\#define vdso_offset_\2 0x\1/p'
```

## Enable USB bluetooth adapter
### Add btusb to kernel
```
$ cat << __EOF__ >> kernel-5.10/arch/arm64/configs/nanopi6_android_defconfig
CONFIG_BT_HCIBTUSB=y
CONFIG_BT_HCIBTUSB_BCM=y
CONFIG_BT_HCIBTUSB_RTL=y
__EOF__
```

### Add bluetooth configuration to Android
```
$ cat << __EOF__ >> device/rockchip/rk3588/nanopi6/BoardConfig.mk 
# Bluetooth
BOARD_HAVE_BLUETOOTH := true
BOARD_BLUETOOTH_BDROID_BUILDCFG_INCLUDE_DIR := device/rockchip/\$(TARGET_BOARD_PLATFORM)/bluetooth

__EOF__
$ vi hardware/interfaces/bluetooth/1.0/default/android.hardware.bluetooth@1.0-service.rc
< service vendor.bluetooth-1-0 /vendor/bin/hw/android.hardware.bluetooth@1.0-service
--
> service vendor.bluetooth-1-0 /vendor/bin/hw/android.hardware.bluetooth@1.1-service.btlinux
$ cat << __EOF__ >> device/rockchip/rk3588/nanopi6/device.mk
PRODUCT_PACKAGES += \\
    android.hardware.bluetooth@1.1-service.btlinux

PRODUCT_COPY_FILES += \\
    frameworks/native/data/etc/android.hardware.usb.host.xml:\$(TARGET_COPY_OUT_VENDOR)/etc/permissions/android.hardware.usb.host.xml

__EOF__
$ echo 'DEVICE_MANIFEST_FILE := device/rockchip/rk3588/nanopi6/manifest.xml' >> device/rockchip/rk3588/nanopi6/device.mk
$ cat << __EOF__ > device/rockchip/rk3588/nanopi6/manifest.xml
<manifest version="1.0" type="device" target-level="6">
    <hal format="hidl">
        <name>android.hardware.audio</name>
        <transport>hwbinder</transport>
        <version>7.0</version>
        <interface>
            <name>IDevicesFactory</name>
            <instance>default</instance>
        </interface>
    </hal>
    <hal format="hidl">
        <name>android.hardware.audio.effect</name>
        <transport>hwbinder</transport>
        <version>7.0</version>
        <interface>
            <name>IEffectsFactory</name>
            <instance>default</instance>
        </interface>
    </hal>
    <hal format="hidl">
        <name>android.hardware.bluetooth</name>
        <transport>hwbinder</transport>
        <version>1.1</version>
        <interface>
            <name>IBluetoothHci</name>
            <instance>default</instance>
        </interface>
    </hal>
    <hal format="hidl">
        <name>android.hardware.graphics.composer</name>
        <transport>hwbinder</transport>
        <version>2.1</version>
        <interface>
            <name>IComposer</name>
            <instance>default</instance>
        </interface>
    </hal>
    <kernel target-level="6"/>
</manifest>
__EOF__
```


## Build
```
$ source build/envsetup.sh && lunch nanopi6-userdebug
$ ./build.sh --all
```

## Burn image to SD card
* Burn official image to SD card
```
$ sudo dd if=/path/to/rk3588-sd-android12-YYYYMMDD.img of=/dev/mmcblk0
```
* Convert simg to img
```
$ simg2img out/target/product/nanopi6/super.img /path/to/super.img
```
* Burn custom images
```
$ sudo dd if=out/target/product/nanopi6/boot.img of=/dev/mmcblk0p7
$ sudo dd if=/path/to/super.img of=/dev/mmcblk0p13
```

# Refer
* https://developer.sony.com/develop/open-devices/guides/aosp-build-instructions/build-aosp-android-12-0
* https://forum.xda-developers.com/t/failed-to-build-arm64-goldfish-kernel-on-mac-os-x.3342461/
* https://source.android.com/docs/devices/automotive/start/passthrough
