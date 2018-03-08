Dax Circuit 03
==============
DAX-C03 is an addon board for the Nvidia Jetson TX1 Development Board that serves as a CAN master node.  It has two MCP2515 CAN controller chips connected to SPI0 and SPI2 on the TX1 Display Expansion connector.  Two ISO1050 isolated CAN tranceivers work as an interface between the bus and the MCP2515.  The MCP2515 is controlled by the MCP251x driver that is included in the linux kernel. 

Status on Jetson TX1:
* Fully working under l4t-28.1 (master)
* Fully working under l4t-24.2 (see releases)

## Table of Contents

1. [Dev Environment Setup](#setup)
2. [Installing the Toolchain](#toolchain)
3. [Add the DTSI](#tx1-dtsi)
6. [Compile the Kernel](#compile)
7. [Flash the TX](#flash)


## Dev Environment Setup <a name="setup"></a>
This section will describe the process of setting up the development environment on a fresh Ubuntu installation using [Jetpack](https://developer.nvidia.com/embedded/jetpack).

First download Jetpack and make the file executable

    chmod +x JetPack-L4T-3.1-linux-x64.run

Next transfer the installer to its own folder.

    mkdir ~/l4t_r28.1
    mv JetPack-L4T-3.1-linux-x64.run ~/l4t_r28.1/

Export ```$DEVDIR``` as the new JetPack location.

    export DEVDIR=~/l4t_r28.1

Now run the installer and follow the instructions.  Once you get to the Package list change the action next to "Install on Target" to "no action."  This will install the dev environment but skip flashing.  We will be modifying the kernel to include this dtsi before flashing the image.

### Installing the Toolchain <a name="toolchain"></a>
In addition to the source you'll need the toolchain for cross-compiling the kernel for ARM.  Download the following two binaries.

* [64bit ARM](https://releases.linaro.org/components/toolchain/binaries/5.4-2017.01/aarch64-linux-gnu/gcc-linaro-5.4.1-2017.01-x86_64_aarch64-linux-gnu.tar.xz)
* [32bit ARM](https://releases.linaro.org/components/toolchain/binaries/5.4-2017.01/arm-linux-gnueabihf/gcc-linaro-5.4.1-2017.01-x86_64_arm-linux-gnueabihf.tar.xz)

Install using the following commands:

    sudo mkdir /opt/linaro
    sudo tar -xf gcc-linaro-5.4.1-2017.01-x86_64_aarch64-linux-gnu.tar.xz -C /opt/linaro/
    sudo tar -xf gcc-linaro-5.4.1-2017.01-x86_64_arm-linux-gnueabihf.tar.xz -C /opt/linaro/

### Add the DTSI <a name="tx1-dtsi"></a>

Download the kernel source code by running the source_sync script.  This will take a few minutes.  Specify which version using the -k tag.

    cd $DEVDIR/64_TX1/Linux_for_Tegra_64_tx1/
    ./source_sync.sh -k tegra-l4t-r28.1

Export ```$SOURCEDIR``` as the sources directory created by ```source_sync.sh```.

    export SOURCEDIR=$DEVDIR/64_TX1/Linux_for_Tegra_64_tx1/sources

Clone the dtsi repository to the sources directory.

    git clone https://github.com/DaxBot/daxc03.git $SOURCEDIR/daxc03

Create symbolic links to the new files and insert them into the kernel.

    # Link dtsi files
    ln -s $SOURCEDIR/daxc03/tegra210-daxc03.dtsi $SOURCEDIR/hardware/nvidia/platform/t210/jetson/kernel-dts/

    # Update device tree to include new entires
    echo "#include \"tegra210-daxc03.dtsi\"" >> $SOURCEDIR/hardware/nvidia/platform/t210/jetson/kernel-dts/tegra210-jetson-tx1-p2597-2180-a01-devkit.dts

### Compile the Kernel <a name="compile"></a>

Install build tools and libraries

    sudo apt update
    sudo apt install -y build-essential libncurses-dev

Export the following environment variables:

    export CROSS_COMPILE=/opt/linaro/gcc-linaro-5.4.1-2017.01-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
    export CROSS32CC=/opt/linaro/gcc-linaro-5.4.1-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc
    export TEGRA_KERNEL_OUT=$SOURCEDIR/compiled

Create the default configuration:

    cd $SOURCEDIR/kernel/kernel-4.4
    mkdir $TEGRA_KERNEL_OUT
    make ARCH=arm64 O=$TEGRA_KERNEL_OUT tegra21_defconfig

Enable driver using menuconfig

    make ARCH=arm64 O=$TEGRA_KERNEL_OUT menuconfig

```bash
Device Drivers --->
    [*] SPI support --->
        <*> User mode SPI device dtsi support
```

```bash
[*] Networking support --->
    <*> CAN bus subsystem support --->
        CAN Device Drivers --->
            <*> Microchip MCP251x SPI CAN controllers
```

Now you can compile the kernel image and device tree blob

    make ARCH=arm64 O=$TEGRA_KERNEL_OUT -j4 zImage dtbs

### Flash the TX <a name="flash"></a>

Copy the new files over to the the Jetpack kernel directory.

    cp $SOURCEDIR/compiled/arch/arm64/boot/Image $SOURCEDIR/../kernel/
    cp $SOURCEDIR/compiled/arch/arm64/boot/zImage $SOURCEDIR/../kernel/
    cp $SOURCEDIR/compiled/arch/arm64/boot/dts/*.dtb $SOURCEDIR/../kernel/dtb/

Update the kernel

    scp $SOURCEDIR/compiled/arch/arm64/boot/Image [user]@[ip]:/boot/

Then restart the TX into recovery mode and flash the DTB

    cd $SOURCEDIR/../
    sudo ./flash.sh -r -k DTB jetson-tx1 mmcblk0p1
