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

Try following this explanation together with cva6 README, I may forgot something.

cva6 Project will be directly pulled in by:

```
git submodule init
git submodules update recursive
```

However, the only thing you may need for running binaries on the soft-cva6 is ```cva6/corev_apu/fpga/ariane.cfg```, which is passed to openocd to connect to the JTAG probe on the FPGA.

## Simulating with Verilator
Although you migh never need it, you can simulate cva6 and binaries with verilator.

This part is Work in progress, but you can follow cva6/README.md to do initialise it.

# Zephyr
This part differs depending on wether you are installing zephyr for the first time or if you have a previous installation that we can modify.
## First installation

You should be able to follow the standard installation process. The only difference to apply is when you run ```west init```

```bash
west init ~/zephyrproject -m https://github.com/Dri99/zephyr-genesysII.git 
```
The only difference is to add the switch ```-m https://github.com/Dri99/zephyr-genesysII.git``` to tell west to use the modified repo as remote.

You don't have to use ```~/zephyrproject``` as destination directory, but it should be consistent with what you chose in previous steps.

Once installation is completed, you can build an application with the switch ```-b genesysII``` to target the new processor.

For a simple test, you can run:
```bash
west build samples/synchronization-sched -b genesysII
```
What we care will be the static executable to run with GDB under ```buid/zephyr/zephyr.elf```.

For a more advance application that can stress the automatic load/store of registers use:
```bash
west build -p always samples/synchronization-sched -b genesysII --build-dir build-sched
```
Note:I prefer to set different build directories for different samples to avoid having to recompile always. You can avoid it.

## Previous installation
In this case you just want to change the remote for the zephyr subdirectory.
Supposing you followed the standard tutorial and successfully installed zephyr in zephyr-project/zephyr

```bash
cd zephyr-project/zephyr
git remote rename origin upstream
git remote add origin https://github.com/Dri99/zephyr-genesysII.git
git pull origin main --set-upstream --rebase
```
What this snippet does is changing the origin repo to my repo, while saving the orginial one as upstream
Then it moves the local main branch to the new origin/main, setting it as new remote branch to receive updates, if any.



# Openocd any version

OpenOCD (On Chip Debugger) is used as common interface between different debuggers (e.g. JLink, Ftdi chips, ...) with GDB. Usually, a probe is connected to the target and communicates with JTAG/SWD. The protocol used to talk with this probe differs by vendor and is sometimes proprietary. OpenOCD supports many of this protocols, allowing a single framework to manage them all, as soon as such probe is supported.

Usually it is present on many package managers. For ubuntu:
```
apt install openocd
```




# How to run
In order to run code on the soft-cva6, you need 2 usb cables connected to the Jtag port and Usart port. The first one is used to program the FPGA and debug code running on the soft-core. The second one shows output.

First step: flash the FPGA
## FPGA bitstream
You Should find some release of CVA6 bitstream on the release page. 
The fpga may come with a preinstalled cva6 inside, but in case you want to flash a different bitstream, you have to: ```Open vivado``` &rarr ```Open Hardware Manager``` $arr ```open target```. At this point you should see xc7k325t in the hardware window; right-click &arr ```Program device``` and select the bitstream you download. Leave anything as it is and click ```Program```.

An alternative flashing procedure can be done with openFPGAloader:

```
openFPGALoader -b genesys2 --bitstream ariane_xilinx.bit
openFPGALoader -b genesys2 -r
```
## Run a binary
With the fpga loaded, you'll need 3 shells for the upcoming commands.
### Shell 1 - Openocd
Just run:
```
openocd -f cva6/corev_apu/fpga/ariane.cfg
```
You can ignore this shell, as soon as it doesn't show any error. Keep the command running

### Shell 2 - Minicom
You can install minicom from apt or any other package manager. It will show the output of ```printf()```s of the program.
To run it, first identify on which tty is writing usart. Usually is goes on ```/dev/ttyUSB0```, but you can double-check with:
```
ls /dev/ttyUSB*
```
To connect to it just:
```
minicom -D /dev/ttyUSBx -b 115200
```
### Shell 3 - GDB

Run the GDB shipped with the toolchain you used. If you used ```riscv-none-elf-gcc``` it should be called ```riscv-none-elf-gdb```

```
riscv-none-elf-gdb executable.elf
```
The executable can be either a program built with cva6-baremetal-bsp or under zephyr/build/zephyr/zephyr.elf

When in gdb connect to the waiting openocd through the socket on port :3333
```
target remote :3333
```
Once done, you can reset the core (only the software, the FPGA stays in place) with ```monitor reset halt```
or ```load``` the binary to it.

Then set some breakpoint (if you want) : ```breakpoint main```

and run ```continue```

For a better list of what you can do with GDB, reference the online guide.

Have fun!!!

