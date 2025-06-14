# UC5_Secure_FPGA_Provisioning
This repository contains code for the end-to-end stack of Use Case 5 of the CROSSCON Project (Intellectual Property Protection for Secure Multi-Tenancy on FPGA) based on Secure FPGA Provisioning.

# Requirements

# Steps

We build the CROSSCON Hypervisor with similar steps as described in https://github.com/crosscon/CROSSCON-Hypervisor-and-TEE-Isolation-Demos/edit/master/zcu-ws/README.md

```sh
export ROOT=`pwd`
```

# Get firmware

Download latest pre-built Zynq UltraScale+ MPSoC Firmware for ZCU102 and use *bootgen*
to build the firmware binary:

```sh
cd zcu-ws
```

```
git clone https://github.com/Xilinx/soc-prebuilt-firmware.git --depth 1 \
    --branch xilinx_v2023.1 firmware

mkdir bin

pushd firmware/zcu102-zynqmp && 
    bootgen -arch zynqmp -image bootgen.bif -w -o ./../../bin/BOOT.BIN
popd

```

# Build optee-os including the TA and pTA

```sh
cd optee_os

OPTEE_DIR="./"
PLATFORM="zynqmp"
PLATFORM_FLAVOR="zcu102"
CC="aarch64-none-elf-"

export CFLAGS=-Wno-cast-function-type
ARCH="arm"
TZDRAM_SIZE="0x00F00000"
SHMEM_SIZE="0x00200000"
CFG_GIC=n

export O="$OPTEE_DIR/optee-aarch64"
TZDRAM_START="0x60000000"
SHMEM_START="0x61000000"

rm -rf $O

make -C $OPTEE_DIR \
    O=$O \
    CROSS_COMPILE=$CC \
    PLATFORM=$PLATFORM \
    PLATFORM_FLAVOR=$PLATFORM_FLAVOR \
    ARCH=$ARCH \
    CFG_PKCS11_TA=n \
    CFG_SHMEM_START=$SHMEM_START \
    CFG_SHMEM_SIZE=$SHMEM_SIZE \
    CFG_CORE_DYN_SHM=n \
    CFG_NUM_THREADS=1 \
    CFG_CORE_RESERVED_SHM=y \
    CFG_CORE_ASYNC_NOTIF=n \
    CFG_TZDRAM_SIZE=$TZDRAM_SIZE \
    CFG_TZDRAM_START=$TZDRAM_START \
    CFG_GIC=$CFG_GIC \
    CFG_ARM_GICV2=y \
    CFG_CORE_IRQ_IS_NATIVE_INTR=n \
    CFG_ARM64_core=y \
    CFG_USER_TA_TARGETS=ta_arm64 \
    CFG_DT=n \
    CFG_CORE_ASLR=n \
    CFG_CORE_WORKAROUND_SPECTRE_BP=n \
    CFG_CORE_WORKAROUND_NSITR_CACHE_PRIME=n \
    CFG_TEE_CORE_LOG_LEVEL=4 \
    CFG_TA_LOG_LEVEL=4 \
    CFG_VULN_PTA=y \
    DEBUG=1 -j16

cd $ROOT
```

# Now we need to build buildroot

Ensure that the CA is rebuilt (if there are changes)

```sh
make O=build-aarch64/ my_test_ca_pkg-rebuild
```

If this is the first time, buildroot config needs the package to be enabled.

Run 


```sh
make menuconfig
```

Go to Target Packages -> Custom Packages -> my_test_ca_pkg and enable it.
If my_test_ca_pkg is not available, check the dependecies in my_test_ca_pkg/Config.in and enable the dependency paackages 

Run

Build buildroot

```sh
make O=build-aarch64/
```

# Build Linux


``` sh
mkdir linux/build-aarch64/
cp support/linux-aarch64.config linux/build-aarch64/.config

cd linux

make ARCH=arm64 O=build-aarch64 CROSS_COMPILE=`realpath ../buildroot/build-aarch64/host/bin/aarch64-linux-` -j16 Image dtbs

cd $ROOT
```

# Bind Linux Image and device tree

```sh
dtc -I dts -O dtb zcu-ws/zcu.dts > zcu-ws/zcu.dtb
```

```sh
cd lloader

rm linux-zcu.bin
rm linux-zcu.elf
make  \
    IMAGE=../linux/build-aarch64/arch/arm64/boot/Image \
    DTB=../zcu-ws/zcu.dtb \
    TARGET=linux-zcu.bin \
    CROSS_COMPILE=aarch64-none-elf- \
    ARCH=aarch64

cd $ROOT
```

# Finally,


``` sh
./build-demo-vtee.sh
```




