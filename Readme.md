The objective of this project is to provide a sort of guide in setting up an environment suitable to develop with Cv64a6.

# Board
The only supported board is the Digilent Genesys II. By using [Litex](), I managed to synthesise it on Digilent arty a7, which could be more comfortable to handle and open (in terms of licenses) to use. However, I never made it to work, so porting it to this FPGA could be doable, but not supported.

# Zephyr port

Much of my work is based on the [WorldofJARcraft's project](https://github.com/WorldofJARcraft/zephyr.git) which is currently being [merged ](https://github.com/zephyrproject-rtos/zephyr/pull/77732) into zephyr. Since in a first time I did't understant the complete series of pull requests and commits from the original project, I copy pasted what I cas needing and ported it in my repository. All credits to @WorldofJARcraft .

For the moment, the only version I ported is the 64 bit one, but from the merge discussion I spotted the 32 bit one too, which is not added to my repo, but which should be easy to do.

To use it, clone my zephyr fork : ```https://github.com/Dri99/cva6.git``` and proceed with the standard installation procedure.

The additional board is called ```genesysII```.

# gcc

This section instructs on how to setup gcc. Usually distro's package managers ships some prebuilt toolchain version, like ```gcc-riscv64-unknown-elf``` on Ubuntu.
Sometimes it might be called riscv-unknown-elf-gcc, or riscv64-unknown-elf-gcc. If it's called riscv32-unknown-elf-gcc it probably only supports 32bit ISA. The important thing is that it supports the ```march=rv64imac_zicsr``` and ```mabi=lp64``` . You can check it with ```riscv{}-unknown-elf-gcc -dumpspecs``` and look for the required ones.
However these versions are often poor of features and additional extensions.

I found complete and prebuilt toolchain, managed by [xpack](https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack). 

Go [this link](https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/releases) and download the latest toolchain riscv-none-elf for linux x86. 

```cd``` to a convenient directory; I put mine in ```$HOME/.local```, but also ```/opt``` is good.
After downloading it, extract with:

```
tar -xvf ~/Downloads/xpack-riscv-none-elf-gcc-14.2.0-3-linux-x64.tar.gz
export PATH="$PWD/xpack-riscv-none-elf-gcc-14.2.0-3/bin:$PATH"
```
Note: your downloaded .tar.gz may be called differently
Note: editing PATH variable is per shell. If you want this command to be permanent put in your ```~/.bashrc``` (with the exact path).

Try building any baremetal application from [cva6-baremetal-bsp](cva6-baremetal-bsp).

If you encounter the error ```relocation truncated to fit: R_RISCV_HI20 against symbol ...``` you are unlucky, but at least you don't have to look for the solution alone. The problem comes from trying to build a 64bit executable. With the bigger addressing space, new relocations are required for jumping between functions in the standard library and the memory model should be different. The standard library is compiled with the memory model ```medlow```, but here we require ```medany```. Unfortunately the majority of precompiled riscv toolchains are compiled with the standard one, so in order to enable ```madany``` we have to recompile gcc.

To do so clone the main repository:
```
git clone https://github.com/riscv-collab/riscv-gnu-toolchain.git
```
Follow the tutorial on the original repository, to install the required dependencies. Then return here for the actual compilation.

```bash
cd riscv-gnu-toolchain
./configure --target=riscv64-unknown-elf --prefix=$HOME/.local --enable-multilib --with-abi=lp64d --with-arch=rv64imafdc --with-cmodel=medany
make newlib
```
The prefix can be any directory (even opt), just ensure you (not root) have write privilege on that one.
This command should compile a riscv64-unknown-elf-gcc library which supports also 32 bit compilation.
remember you can add ```-j$NPROC``` to build on more cores and just be patient.

When finished remember to add HOME/.local/bin to your PATH environment variable.

# cva6-baremetal-bsp

Project forked from openhwgroup with some baremetal applications useful for testing cva6 features without a fully featured OS like Zephyr.

To build any app, go in the [app](cva6-baremetal-bsp/app) foldes and use

```bash
make all XLEN=64
```
If your toolchain is riscv-unknown... you can build with
```bash
make all XLEN=""
```
If interested in only building a single app you can set the variable ```bmarks``` like
```bash
make all XLEN=64 bmarks=soc_context_test
```
Note: The makefile is defined to use riscv-none-elf-gcc, like explained in the gcc section. If you manage to install your gcc version, nothing stops you from setting your own. The xpack version has proved to work.

Indeed, soc_context_test, helloworld, and helloworld_printf are the only ones I tested.

If you encounter the infamous error ```relocation truncated to fit: R_RISCV_HI20 against symbol``` it probably means that you have to recompile gcc. The explanation is in gcc section.

# Vivado

Vivado version 2020.2 . The choice is due to this version being the last one compatible with the license I owned. It should support newer versions but mind the licenses before installing it. The best should be to use an FPGA thad doesn't require licensing, like the Arty A7.

https://www.xilinx.com/member/forms/download/xef.html?filename=Xilinx_Unified_2020.2_1118_1232.tar.gz

Unfortunately, the only option is to download a single big file as the online installer (which downloads a small program from browser and then manages the download on its own) is not anymore supported.

While installing vivado, you can reduce the total install size by opting out the majority of supported FPGAs. Those we might be interested in are 

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

