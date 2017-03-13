Dax Circuit 03
==============
DAXC03 serves as the CAN master node.  It has two MCP2515 CAN controller chips connected to SPI0 and SPI2 on the TX1 Display Expansion connector.  Two ISO1050 isolated CAN tranceivers work as an interface between the bus and the MCP2515.  The MCP2515 is controlled by the MCP251x driver that is included in the linux kernel (http://lxr.free-electrons.com/source/drivers/net/can/mcp251x.c?v=3.13).

Kernel Setup
============
In order for the MCP2515 to work the kernel needs to have both SPI and the MCP251x driver enabled in the kernel.  First step is becoming familiar with the process for building the kernel.  Ridgerun has an excellent guide on the subject here: https://developer.ridgerun.com/wiki/index.php?title=Compiling_Tegra_X1_source_code

Set CONFIG_SPI_SPIDEV=y
```bash
Device Drivers --->
    [*] SPI support --->
        <*> User mode SPI device driver support
```

Set CONFIG_CAN_DEV=y, CONFIG_CAN_MCP251X=y
```bash
[*] Networking support --->
    <*> CAN bus subsystem support --->
        CAN Device Drivers --->
            <*> Microchip MCP251x SPI CAN controllers
```

Device Tree
===========
SPIDEV is enabled in the device tree.  Modifications need to be included in the compiled tree (blob) that is read in by the TX1.
```bash
demsg | grep .dts
[    0.000000] DTS File Name: arch/arm64/boot/dts/tegra210-jetson-tx1-p2597-2180-a01-devkit.dts
```

This can be done by including tegra210-daxc03.dtsi in the main dts file arch/arm64/boot/dts/tegra210-jetson-tx1-p2597-2180-a01-devkit.dts before the blob is compiled (make dtbs).  The result can then be transferred to /boot/ on the TX1.

Alternatively the blob can be decompiled, modified, and recompiled directly on the TX1.  This method is described here: http://elinux.org/Jetson/TX1_SPI

Board File
==========
The version of the MCP251x driver included in the kernel does not use the device tree for its configuration.  Thus the TX1's board file needs to be modified to add the driver's private data structs.

This board file compiled into the TX1 kernel is located here:
```
kernel_source/arch/arm64/mach-tegra/board-t210ref.c
```
