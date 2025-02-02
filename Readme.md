The objective of this project is to provide a sort of guide in setting up an environment suitable to develop with Cv64a6.

# Table of contents
- [Introduction](#introduction)
    1. [Board](#board)
    2. [Zephyr port](#zephyr-port)
    2. [Hardware Customisations](#hardware-customisations)
        - [Store](#automatic-context-store)
        - [Load](#automatic-context-load)
        - [CSRs](#additional-csrs)
- [Installation](#installation)
    1. [Clone this repo](#repo-setup)
    2. [Install gcc](#install-gcc)
    3. [Install zephyr](#install-zephyr)
    4. [Install Vivado](#install-vivado)
    5. [OpenOCD](#openocd-any-version)
- [Building](#building-applications)
    - [Baremetal apps](#baremetal-applications)
    - [Zephyr apps](#zephyr-applications)
- [Running](#running-applications)
    1. [Load FPGA](#fpga-bitstream)
    2. [openOCD](#shell-1---openocd)
    3. [minicom](#shell-2---minicom)
    4. [GDB](#shell-3---gdb)


# Introduction 
## Board
The only supported board is the Digilent Genesys II. By using [Litex](https://github.com/enjoy-digital/litex), I managed to synthesise cva6 on Digilent arty a7, which could be more comfortable to handle and open (in terms of licenses) to use. 
However, I never made it to work run anything on it, getting stuck at failing in the JTAG connection.
Nonetheless, Arty A7 support for the cva6 should be doable either in the same way openhwgroup supports previous boards, or by digging more litex setups.

## Zephyr port

Much of my work is based on the [WorldofJARcraft's project](https://github.com/WorldofJARcraft/zephyr.git) which is currently being [merged ](https://github.com/zephyrproject-rtos/zephyr/pull/77732) into zephyr. Since in a first time I did't understand the complete series of pull requests and commits from the original project, I copy pasted what I needing and ported it in my repository. All credits to @WorldofJARcraft .

For the moment, the only version I ported is the 64 bit one, but from the pull request discussion I spotted the 32 bit one too, which is not added to my repo, but which should be easy to do.

The additional board is called ```genesysII```.

## Hardware customisations

The original core cva6 has been modified mainly in the issue stage, adding a Shadow Register Unit (SHRU) at the level of the register file, to increase management speed of interrupts. This work is higly inspired to [CV32RT](https://ar5iv.labs.arxiv.org/html/2311.08320#S3.SS1) which was brought on the CVA4. The increased complexity of its successor forced me to apply some differences, especially in the communication between the SHRU and the memory, that here happens through the Cache, and in the communication with the CSR Register File.

### Automatic Context Store

Whenever any interrupt is taken, the SHRU makes a copy of the caller saved registers and allocates the space on the stack decreasing ```sp```. The only exceptions not triggering this mechanism are debug exceptions (Debug request and Breakpoint), as their management jumps to a portion of code not directly managed by software (usually, the debug module). 

In following cycles, the SHRU performs writes to memory (through the cache), without stealing bandwidth to the Load Store Unit (LSU). The algorithm happens concurrently to the normal execution, so that the interrupt can be served as soon as possible.

An additional CSR is supplied, CSR_LAST_SP, which stores the ```sp``` value of the last context store issue by the SHRU. While not strictly required for correct functioning of the unit, it can be useful in debugging interrupt misbehaviours.

### Automatic Context Load

Regarding the load, some work has to be done in advance by software. CSR_LOAD_ESF must be written with the base address of the stack frame we will want to read from, and when we are sure we will use it (any scheduling decision has been made) a 1 must be written to CSR_SHADOW_STATUS.start_load. This will trigger the load mechanism from RAM, which will be brought concurrently to program execution. 

When the processor commits an mret, values loaded from RAM and waiting in the shadow registers get written all at once in to the architectural registers. 

Note that if the loading hasn't concluded, the processor will stall at the mret level, to ensure all registers contains the expected values after mret execution. This brings a limitation in the efficiency of using sich a solution. In case we may know the next process to execute only few instructions before the return from interrupt, the time taken will be the same as without this optimisation. 

Nevertheless, this case is the worst case, meaning that the feature can only improve performance and at worse give the same, but never be detrimental.


Thus, for an automatic load to be effective, the earlier we know the next process in execution, the better it is for performance. 

In theory, in Zephyr the scheduling decision for the next process to execute is know even before the current process to run is started. This should happen because when a process is put in run state, it is removed from the run queue and the next on is immediately selected. This should allow to start the load process even before the current process is saved, but it's application has not been studies. At the moment my code in zephyr doesn't use efficiently this optimisation, starting the save very late.

### Additional CSRs

- CSR_LAST_SP (0x7C6)

    Read-Only: shows base address of the last automatic save, even when such a save is still running

- CSR_SHADOW_STATUS (0x7C7)

  It is split in many parts, some of which read-only and others write only
  - start_load: starts restoring registers from RAM at the stack frame at CSR_LOAD_ESF to the shadow registers. After a 1 is written to it, it freezes the value in CSR_LOAD_ESF until the restore has complete. (Write only)
  - save_ready: the SHRU is not running a context save, so it can treat a new interrupt. (Read-only)
  - load_level: registers not yet restores. It counts backwards from 16 down to 0. At 0 the mret can commits.
  - raddr : sets the shadown register address read by CSR_SHADOW_REG. The typical access type is to write an address (0:15) into CSR_SHADOW_STATUS.raddr  and then read CSR_SHADOW_REG. At the moment, only shadow registers used for automatic store are shown, but it shouldn't be hard to add support for reading also load shadow registers. (RW)
  - save_level: same as load_level, but it shows progress in current store. It decrement from 15 to 0. (Red only)


|31 . . . 27|      26    | 25 |     24     | 23 . . . . 21| 20 . . . 16 | 15 . . . . 13 | 12 . . 8 | 7 . . . 5 | 4 . . . . 0 |
| :-------: | :--------: |:-: | :--------: | :----------: | :--------:  | :----------:  | :------: | :--------:| :---------: |
| Reserved  | start_load | 0  | save_ready |  Reserved    | load_level  |  Reserved     | raddr    | Reserved  | save_level  |
|      5    |      1     | 1  |     1      |  3           |  5          | 3             | 5        | 3         | 5           |

- CSR_SHADOW_REG (0x7C8)
    Shows the content of shadow register addressed by CSR_SHADOW_STATUS.raddr, even after context store has concluded. See CSR_SHADOW_STATUS for more details. (Read Only)

- CSR_LOAD_ESF (0x7C9)
    Exception Stack Frame from which to load next context restore. See CSR_SHADOW_STATUS for more details. (RW). The structure of the ESF is that of an array of 16 uint64_t or :
    ```C
    struct esf {
		unsigned long ra;
		unsigned long t0;
		unsigned long t1;
		unsigned long t2;
		unsigned long a0;
		unsigned long a1;
		unsigned long a2;
		unsigned long a3;
		unsigned long a4;
		unsigned long a5;
		unsigned long a6;
		unsigned long a7;
		unsigned long t3;
		unsigned long t4;
		unsigned long t5;
		unsigned long t6;
	};
    ```
    As any C structure, first member are stored in the smallest address, so that CSR_LOAD_ESF+0 contains ```$ra``` and CSR_LOAD_ESF+120 contains ```t6```.

# Installation

## Repo setup
The only important thing, since this repo makes use of git submodules, is to run:
```bash
git submodule init
git submodule update --recursive
```

Every now and then, the submodules may update. It is in general safe to update this repo too by issuing:
```bash
git submodule update --remote --recursive
```

Anyway it is not mandatory to use submodules and have the mentioned gits elsewhere (that's the setup I used), but in this way it is easier to refer to scripts in this tutorial.

## Install GCC

This section instructs on how to setup gcc. Usually distro's package managers ships some prebuilt toolchain version, like ```gcc-riscv64-unknown-elf``` on Ubuntu.
Sometimes it might be called riscv-unknown-elf-gcc, or riscv64-unknown-elf-gcc. If it's called riscv32-unknown-elf-gcc it probably only supports 32bit ISA. The important thing is that it supports the ```march=rv64imac_zicsr``` and ```mabi=lp64``` . You can check it with ```riscv{}-unknown-elf-gcc -dumpspecs``` and look for the required ones.
However these versions are often poor of features and additional extensions.

I found complete and prebuilt toolchain, managed by [xpack](https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack). 

Go to [this link](https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/releases) and download the latest toolchain riscv-none-elf for linux x86. 

Choose a convenient directory for installing ; I put mine in ```$HOME/.local```, but also ```/opt``` is good.
After downloading it, extract with:

```bash
cd $GCC_INSTALL_PATH
wget https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/releases/download/v14.2.0-3/xpack-riscv-none-elf-gcc-14.2.0-3-linux-arm.tar.gz
tar -xvf ./xpack-riscv-none-elf-gcc-14.2.0-3-linux-x64.tar.gz
export PATH="$GCC_INSTALL_PATH/xpack-riscv-none-elf-gcc-14.2.0-3/bin:$PATH"
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

## Install zephyr

This part differs depending on wether you are installing zephyr for the first time or if you have a previous installation that we can modify.

### First installation

You should be able to follow the standard installation process. The only difference to apply is when you run ```west init```

```bash
west init ~/zephyrproject -m https://github.com/Dri99/zephyr-genesysII.git 
```
The only difference is to add the switch ```-m https://github.com/Dri99/zephyr-genesysII.git``` to tell west to use the modified repo as remote.

You don't have to use ```~/zephyrproject``` as destination directory, but it should be consistent with what you chose in previous steps.

### Previous installation

In this case you just want to change the remote for the zephyr subdirectory.
Supposing you followed the standard tutorial and successfully installed zephyr in zephyr-project/zephyr

```bash
cd $ZEPHYR_LOCATION/zephyr
git remote rename origin upstream
git remote add origin https://github.com/Dri99/zephyr-genesysII.git
git pull origin main --set-upstream --rebase
west update
```
What this snippet does is changing the origin repo to my repo, while saving the orginial one as upstream
Then it moves the local main branch to the new origin/main, setting it as new remote branch to receive updates, if any.

Sometimes, especially if you were on a main version newer than the one on which I was based, it might take newer commits that I could not support.
You can spot it by seeing with ```git log``` that the last commit is not mine. 

In that case, if you are sure you saved anything in the repo online or in a safe place, you can try
```sh
git reset --hard origin/main
```
Pay attention: ALL YOUR DATA WILL BE ERASED on the local main branch.


## Install Vivado

Vivado version 2020.2 . The choice is due to this version being the last one compatible with the license I owned. It should support newer versions but mind the licenses before installing it. The best should be to use an FPGA thad doesn't require licensing, like the Arty A7.

https://www.xilinx.com/member/forms/download/xef.html?filename=Xilinx_Unified_2020.2_1118_1232.tar.gz

Unfortunately, the only option is to download a single big file as the online installer (which downloads a small program from browser and then manages the download on its own) is not anymore supported.

While installing vivado, you can reduce the total install size by opting out the majority of supported FPGAs. Those we might be interested in are 


## Openocd any version

OpenOCD (On Chip Debugger) is used as common interface between different debuggers (e.g. JLink, Ftdi chips, ...) with GDB. Usually, a probe is connected to the target and communicates with JTAG/SWD. The protocol used to talk with this probe differs by vendor and is sometimes proprietary. OpenOCD supports many of this protocols, allowing a single framework to manage them all, as soon as such probe is supported.

Usually it is present on many package managers. For ubuntu:
```bash
apt install openocd
```

# Building applications

## Baremetal applications

Project forked from openhwgroup with some baremetal applications useful for testing cva6 features without a fully featured OS like Zephyr.

To build any app, go in the [app](cva6-baremetal-bsp/app) foldes and use

```bash
make all
```
Yes, I fixed the Makefile to build all easily buildable apps at once. Dhrystone and pmp are not part of this set.
However, only few of this may be interesting.
If interested in only building a single app you can set the variable ```bmarks``` like
```bash
make all bmarks=soc_context_test
```
Note: The makefile is defined to use riscv-none-elf-gcc, like explained in the gcc section. If you manage to install your gcc version, nothing stops you from setting your own. The xpack version has proved to work.

Indeed, soc_context_test, helloworld, and helloworld_printf are the only ones I tested.

If you encounter the infamous error ```relocation truncated to fit: R_RISCV_HI20 against symbol``` it probably means that you have to recompile gcc. The explanation is in gcc section.


## Zephyr applications

You should be able to create out-of-tree zephyr apps, like explained at [this link](https://docs.zephyrproject.org/latest/develop/application/index.html). It is in general preferable to follow it, to keep zephyr sources as clean as possible (Note that it's not what I did).

To build with my modified version of zephry you just have to source ```zephyr-env.sh``` in the zephry src tree

For example:
```bash
source $ZEPHYR_LOCATION/zephyr/zephyr-env.sh
source $ZEPHYR_LOCATION/.venv/bin/activate
```

This will allow to the current shell to know where zephyr is and activate zephyr virtual environment, to be able to use west.

Once installation is completed, you can build an application with the switch ```-b genesysII``` to target the new processor.


```sh
west build -b gensysII $APP_DIR
```

For a simple test, you can run:
```bash
west build samples/helloworld -b genesysII
```
What we care will be the static executable to run with GDB under ```buid/zephyr/zephyr.elf```.

For a more advance application that can stress the automatic load/store of registers use:
```bash
west build -p always samples/synchronization-sched -b genesysII --build-dir build-sched
```
Note:I prefer to set different build directories for different samples to avoid having to recompile always. You can avoid it.


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

# Running applications
In order to run code on the soft-cva6, you need 2 usb cables connected to the Jtag port and Usart port. The first one is used to program the FPGA and debug code running on the soft-core. The second one shows output.

First step: flash the FPGA
## FPGA bitstream

### Vivado
You Should find some release of CVA6 bitstream on the release page. 
The fpga may come with a preinstalled cva6 inside, but in case you want to flash a different bitstream, you have to: ```Open vivado``` &rarr; ```Open Hardware Manager``` &rarr; ```open target```. At this point you should see xc7k325t in the hardware window; right-click &rarr; ```Program device``` and select the bitstream you download. Leave anything as it is and click ```Program```.

### openFPGAloader
An alternative flashing procedure can be done with openFPGAloader:

```sh
openFPGALoader -b genesys2 --bitstream $DOWNLOAD_LOCATION/ariane_xilinx.bit
openFPGALoader -b genesys2 -r
```

## Shell 1 - Openocd
With the fpga loaded, you'll need 3 shells for the upcoming commands.

Just run:
```
openocd -f cva6/corev_apu/fpga/ariane.cfg
```
You can ignore this shell, as soon as it doesn't show any error. Keep the command running

## Shell 2 - Minicom
You can install minicom from apt or any other package manager. It will show the output of ```printf()```s of the program.
To run it, first identify on which tty is writing usart. Usually is goes on ```/dev/ttyUSB0```, but you can double-check with:
```
ls /dev/ttyUSB*
```
To connect to it just:
```
minicom -D /dev/ttyUSBx -b 115200
```
## Shell 3 - GDB

Run the GDB shipped with the toolchain you used. If you used ```riscv-none-elf-gcc``` it should be called ```riscv-none-elf-gdb```

```
riscv-none-elf-gdb <executable.elf>
```
The executable can be either a program built with cva6-baremetal-bsp or under zephyr/build/zephyr/zephyr.elf

When in gdb connect to the waiting openocd through the socket on port :3333
```
target remote :3333
```
Once done, you can reset the core (only the software, the FPGA stays in place) with ```monitor reset halt```
or ```load``` the binary to it. My advise is to always reset and load, as sometimes the softcore gets in strange states.

Then set some breakpoint (if you want) : ```breakpoint main```

and run ```continue```

For a better list of what you can do with GDB, reference the online guide.

Have fun!!!

