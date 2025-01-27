The objective of this project is to provide a sort of guide in setting up an environment suitable to develop with Cv64a6.

# Board
The only supported board is the Digilent Genesys II. By using [Litex](), I managed to synthesise it on Digilent arty a7, which could be more comfortable to handle and open (in terms of licenses) to use. However, I never made it to work, so porting it to this FPGA could be doable, but not supported.

# Zephyr port

Much of my work is based on the [WorldofJARcraft's project](https://github.com/WorldofJARcraft/zephyr.git) which is currently being [merged ](https://github.com/zephyrproject-rtos/zephyr/pull/77732) into zephyr. Since in a first time I did't understant the complete series of pull requests and commits from the original project, I copy pasted what I cas needing and ported it in my repository. All credits to @WorldofJARcraft .

For the moment, the only version I ported is the 64 bit one, but from the merge discussion I spotted the 32 bit one too, which is not added to my repo, but which should be easy to do.

To use it, clone my zephyr fork : ```https://github.com/Dri99/cva6.git``` and proceed with the standard installation procedure.

The additional board is called ```genesysII```.

# gcc

https://github.com/riscv-collab/riscv-gnu-toolchain.git

A first point is to rebuild gcc from sources, changing an option.

The problem is due to using 64 bit processor. I chose the 64 bit version because the partial porting (of cva6 to Zephyr) I borrowed used this version. I 

# Vivado

Vivado version 2020.2 . The choice is due to this version being the last one compatible with the license I owned. It should support newer versions but mind the licenses before installing it. The best should be to use an FPGA thad doesn't require licensing, like the Arty A7.

https://www.xilinx.com/member/forms/download/xef.html?filename=Xilinx_Unified_2020.2_1118_1232.tar.gz

Unfortunately, the only option is to download a single big file as the online installer (which downloads a small program from browser and then manages the download on its own) is not anymore supported.

# Cva6 Project

follow steps on cva6 project

# Openocd any version

OpenOCD (On Chip Debugger) is used as common interface between different debuggers (e.g. JLink, Ftdi chips, ...) with GDB. Usually, a probe is connected to the target and communicates with JTAG/SWD. The protocol used to talk with this probe differs by vendor and is sometimes proprietary. OpenOCD supports many of this protocols, allowing a single framework to manage them all, as soon as such probe is supported.


# How to run

Openocd (can be ignored, apart if it gives errors)

minicom to read serial output

gdb executable.elf

target remote :3333

monitor reset halt

load

breakpoint (if any)

continue

