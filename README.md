# Instructions for Building the Symbian EKA2 Kernel

This repository contains instructions on how to build the now defunct Symbian operating system's second-generation (EKA2) kernel.

A virtualized instance of Ubuntu Linux 10.04 (32-bit) running on VirtualBox will be necessary, as most of the Symbian build tools for Linux were intended to run on legacy versions from over a decade ago. You can obtain the ISO image for it [here](http://old-releases.ubuntu.com/releases/10.04.0/ubuntu-10.04.4-desktop-i386.iso).

*N.B. While there is support for the 64-bit version of Ubuntu Linux 10.04, it requires additional steps which may complicate the overall build process and thus will not be covered in this guide.*

These instructions were tested on VirtualBox Version 6.1.38 r153438.

## 1. Initial Configuration

The files required are available at the following locations:

* Siemens Website - [Sourcery G++ Lite for SymbianOS](https://sourcery.sw.siemens.com/GNUToolchain/package6500/public/arm-none-symbianelf/arm-2010q1-190-arm-none-symbianelf.bin)
* Internet Archive - [PDK 4.0a](https://archive.org/download/nokia_sdks_n_dev_tools/PDK_4_0a.zip)
* GitHub - [Kernel (GCC_SURGE Branch)](https://github.com/SymbianSource/oss.FCL.sf.os.kernelhwsrv/tree/BRANCH_GCC_SURGE)
* GitHub - [QEMU (GCC_SURGE Branch)](https://github.com/SymbianSource/oss.FCL.sf.adapt.Qemu/tree/BRANCH_GCC_SURGE)
* GitHub - [Linux Build Tools](https://github.com/SymbianSource/oss.FCL.interim.SFTools.LinuxBuild)

After installing Ubuntu 10.04 in VirtualBox, import the aforementioned files into the VM. Place them in a convenient location, preferably `Documents` or `Downloads`.

Ubuntu's repositories contain a fair amount of prerequisite software for building the kernel packages. Since this is a legacy version, `/etc/apt/sources.list` should be updated to retrieve packages from the `old-release` repositories:

```console
$ sudo sed -i -re 's/([a-z]{2}\.)?archive.ubuntu.com|security.ubuntu.com/old-releases.ubuntu.com/g' /etc/apt/sources.list
```

Now, update the package list and fetch the latest version available for existing system packages.

```console
$ sudo apt-get update && sudo apt-get dist-upgrade
```

Symbian packages tended to be archived in the 7z format. The `p7zip` program can deal with such archives.

```console
$ sudo apt-get install p7zip-full
```

The EKA2 build environment can now be configured. Onward!

## 2. Setting up the EKA2 Build Tools

When working through this section, refer to the directory where you imported the files in the VM.

### CodeSourcery ARM GCC Toolchain

Prior to starting the CodeSourcery installer, switch the system shell from Ubuntu's default Dash to Bash, as needed by the installer program. When prompted to install Dash as `/bin/sh`, select 'No'.

```console
$ sudo dpkg-reconfigure -plow dash
```

Start the installer.

```console
$ ./arm-2010q1-190-arm-none-symbianelf.bin
```

In the installer, ensure the following:

* Your home folder is the install path i.e., `/home/yourusernamehere/CodeSourcery/Sourcery_G++_Lite`
* The 'Modify PATH for current user' option is selected
* When asked about shortcuts and link creation, the 'Other' option is selected and your home folder is set as the path i.e., `/home/yourusernamehere/CodeSourcery/Sourcery_G++_Lite_for_ARM_SymbianOS`

After the installation is complete, give it a second or two to fully return control back to the shell.

Finally, create a softlink to the toolchain's install location.

```console
$ ln -s ~/CodeSourcery/Sourcery_G++_Lite ~/gcce
```

### Raptor (Symbian Build System Version 2)

At the time, Raptor (also referred to as SBSv2) was intended to be a new cross-platform build system capable of building the entire platform with ease. In our case, it needs to be built from source and bootstrapped for use.

First, install the prerequisite packages for Raptor:

```console
$ sudo apt-get install build-essential bison ncurses-dev libbz2-dev libboost1.40* libxml2 libxml2-dev
```

Before proceeding further, an **EPOCROOT** directory must be created. This is essentially a build directory where the `epoc32` binaries are populated during the build process, including the final ROM images for the kernel.

```console
$ mkdir ~/symbian # This is our EPOCROOT
```

Depending on where you placed the imported files, unpack the `oss.FCL.interim.SFTools.LinuxBuild-master.zip` archive, rename it to `linux_build`. Then, move it to the `EPOCROOT` directory. This pattern will be followed for subsequent packages.

```console
$ 7z x oss.FCL.interim.SFTools.LinuxBuild-master.zip
$ mv oss.FCL.interim.SFTools.LinuxBuild-master linux_build
$ mv linux_build ~/symbian
```

Execute the Raptor build script. Sit back and relax as it takes a while.

```console
$ cd ~/symbian/linux_build/cross-plat-dev-utils
$ ./build_raptor.pl
```

## 3. Product Development Kit (PDK)

Symbian Product Development Kits were intended for platform developers, usually OEMs to modify the OS for their interests. PDKs typically shipped with the Symbian source code and tons of prebuilt binaries. In this case, the latter is of particular interest to us.

After unpacking the `PDK_4_0a.zip` archive, move **only** `binaries_epoc.7z.zip` & `tools_epoc.7z.zip` into the `EPOCROOT` directory. 

```console
$ 7z x PDK_4_0a.zip
$ cd pdk/PDK_4.0.a
$ mv binaries_epoc.7z.zip tools_epoc.7z.zip ~/symbian
```

Unpack the newly-copied archives to begin populating the `epoc32` tree.

```console
$ cd ~/symbian
$ for file in *.zip; do 7z x $file; done
```

### Fix the EPOC32 tree & Raptor Configuration

There are some inherent bugs in the Raptor GCC runtime configuration that need to be patched specifically for Linux. 

```console
$ cd ~/symbian/linux_build/cross-plat-dev-utils
$ ./prep_env.pl
```

This command makes the following changes:

* The host GCC standard library is selected as the Standard C++ library instead of the STLport 5.1 implementation.
* GCC is configured to compile to the C++0x standard.
* Install a GCC pre-include header in `epoc32/include/gcc` to incorporate GCC 4.4.x compatiblity.

Finally, build all the supported targets in the directory.

```console
$ cd ~/symbian/linux_build/cross-plat-dev-utils
$ ./build_all.pl
```

## 3. QEMU

In the later stages of Symbian's short-lived open source chapter, a novel means of emulating ARM hardware boards was introduced - The Symbian Virtual Platform (SVP). It provides a virtual board model called "Syborg" that is seamlessly integrated with QEMU's ARM system emulator target.

Begin by first installing all the QEMU dependencies.

```console
$ sudo apt-get install libexpat1 libexpat1-dev zlib1g libpng12-0 libsdl1.2debian libsdl1.2-dev python-dev
```

Create a directory within `EPOCROOT` to build QEMU.

```console
$ cd ~/symbian
$ mkdir -p sf/adapt
```

Unpack the `oss.FCL.sf.adapt.Qemu-BRANCH_GCC_SURGE.zip` archive, rename it to `qemu` and move the extracted folder to `sf/adapt`. 

```console
$ 7z x oss.FCL.sf.adapt.Qemu-BRANCH_GCC_SURGE.zip
$ mv oss.FCL.sf.adapt.Qemu-BRANCH_GCC_SURGE qemu
$ mv qemu ~/symbian/sf/adapt
```

Now, make a shell script called `qemu_build.sh`.

```console
$ cd ~/symbian
$ touch qemu_build.sh
```

Paste the code below into the script.

```shell
#!/bin/sh
# Pass 'conf' to ./configure. Otherwise, pass 'clean' to clean.
export EPOCROOT=~/symbian/
export SBS_HOME=${EPOCROOT}linux_build/sbsv2/raptor
export SBS_GCCE441BIN=~/gcce/bin
export PYTHONPATH=${EPOCROOT}sf/adapt/qemu/symbian-qemu-0.9.1-12/qemu-symbian-svp/plugins
cd ${EPOCROOT}sf/adapt/qemu/symbian-qemu-0.9.1-12/qemu-symbian-svp
if [ "$1" == "conf" ]; then
chmod +x configure
./configure --target-list=arm-softmmu --prefix=${EPOCROOT}sf/adapt/qemu/symbian-qemu-0.9.1-12
fi
if [ "$1" == "clean" ]; then
make clean
else
make && make install
fi
```

Run the script to configure and build QEMU.

```console
$ chmod +x qemu_build.sh
$ ./qemu_build.sh conf
```

## 4. Building the Kernel & Syborg Boot ROM

At last, the section long awaited for. Everything is in place to proceed with finally building the kernel.

### Kernel

Set up a directory within `EPOCROOT` for the `kernelhwsrv` package.

```console
$ cd ~/symbian
$ mkdir -p sf/os
```

Unpack the `oss.FCL.sf.os.kernelhwsrv-BRANCH_GCC_SURGE.zip` archive, rename it to `kernelhwsrv` and move the extracted folder to `sf/os`. 

```console
$ 7z x oss.FCL.sf.os.kernelhwsrv-BRANCH_GCC_SURGE.zip
$ mv oss.FCL.sf.os.kernelhwsrv-BRANCH_GCC_SURGE kernelhwsrv
$ mv kernelhwsrv ~/symbian/sf/os
```

Now, create the script `ksrt_build.sh` to build the kernel-side runtime library.

*N.B. This script needs to be run first, since the GCCE kernel-side runtime `ksrt_gcce.lib` must be available to compile the actual kernel.*

```console
$ cd ~/symbian
$ touch ksrt_build.sh
```

Paste the code below into the script.

```shell
#!/bin/sh
export EPOCROOT=~/symbian/
export SBS_HOME=${EPOCROOT}linux_build/sbsv2/raptor
export SBS_GCCE441BIN=~/gcce/bin
cd ${EPOCROOT}sf/os/kernelhwsrv/kernel/eka/compsupp/gcce
$SBS_HOME/bin/sbs -c arm.v5.udeb.gcce4_4_1.surge -k $1
```

Execute the script.

```console
$ chmod +x ksrt_build.sh
$ ./ksrt_build.sh
```

Naturally, the kernel is our next target. Create the script `kernel_build.sh`.

```console
$ cd ~/symbian
$ touch kernel_build.sh
```

Paste the code below into the script.

```shell
#!/bin/sh
export EPOCROOT=~/symbian/
export SBS_HOME=${EPOCROOT}linux_build/sbsv2/raptor
export SBS_GCCE441BIN=~/gcce/bin
cd ${EPOCROOT}sf/os/kernelhwsrv
$SBS_HOME/bin/sbs -s package_definition.xml -c arm.v5.udeb.gcce4_4_1.surge -k -j 4 $1
```

Execute the script.

```console
$ chmod +x kernel_build.sh
$ ./kernel_build.sh
```

### Syborg

A text-shell Syborg ROM based on the built kernel can be generated. Create the script `syborg_build.sh`

```console
$ cd ~/symbian
$ touch syborg_build.sh
```

Paste the code below into the script.

```shell
#!/bin/sh
export EPOCROOT=~/symbian/
export SBS_HOME=${EPOCROOT}linux_build/sbsv2/raptor
export SBS_GCCE441BIN=~/gcce/bin
cd  ${EPOCROOT}sf/adapt/qemu/baseport/syborg
$SBS_HOME/bin/sbs -c arm.v5.udeb.gcce4_4_1.surge $1
```

Execute the script.

```console
$ chmod +x syborg_build.sh
$ ./syborg_build.sh
```

Assuming everything went smoothly, the Syborg ROM image is produced as `epoc32/rom/syborg_tshell_ARMV5_udeb.img`.

Additionally, to assist with debuggging, the following files are produced at `epoc32/rom/syborg` -

* `syborg_tshell_ARMV5_udeb_ROMBUILD.LOG`
* `syborg_tshell_ARMV5_urel_rom.oby`
* `syborg_tshell_ARMV5_udeb.symbol`

### Booting

Last but not the least, it's time to boot the ROM file. To make the process straightforward, create a script `boot_rom.sh`

```console
$ cd ~/symbian
$ touch boot_rom.sh
```

Paste the code below into the script.

```shell
#!/bin/sh
export EPOCROOT=~/symbian/
export PYTHONPATH=${EPOCROOT}sf/adapt/qemu/symbian-qemu-0.9.1-12/qemu-symbian-svp/plugins
cd ${EPOCROOT}sf/adapt/qemu/symbian-qemu-0.9.1-12/bin
./qemu-system-arm -serial file:${EPOCROOT}syborg_serial.log \
-M ${EPOCROOT}sf/adapt/qemu/baseport/syborg/syborg.dtb -kernel ${EPOCROOT}epoc32/rom/syborg_tshell_ARMV5_udeb.img
```

Execute the script.

```shell
$ chmod +x boot_rom.sh
$ ./boot_rom.sh
```

QEMU should boot the Symbian text-shell ROM accompanied with intermittent console messages. All kernel tracing output is written to `${EPOCROOT}syborg_serial.log` 

```
syborg_nvmemorydevice: create

drive size:  67108864
sector size:  512
drive name:  syborg_system.img
syborg_nvmemorydevice: Check drive image path

syborg_nvmemorydevice: drive image not found - create

array length:  524288
imagepath:  /home/yukihyou6840/symbian/sf/adapt/qemu/symbian-qemu-0.9.1-12/bin/nvmemory/syborg_system.img
handle created: 0
use handle: 0
syborg_nvmemorydevice: created

syborg_nvmemorydevice: host addr: 0x96939000
```

This concludes our journey in building a rudimentary text-shell ROM for the Symbian kernel.

## Bibliography

Instructions presented here were adapted from the Symbian Foundation's wiki and its mirrors' archived pages on the Wayback Machine. 

Most of these sources present a rather fragmented picture of how to build the platform, and refer to web links that have not been archived. Moreover, when the Symbian Developer website was completely taken down around 2011, ample useful information was lost with it.

### Links

1. [How to build and run a Symbian QEMU ROM with GNU/Linux Tools](https://web.archive.org/web/20210507210415/https://akawolf.org/wiki/index.php/How_to_build_and_run_a_Symbian_QEMU_ROM_with_GNU/Linux_tools)

2. [Bootstrapping the Symbian Build Tools on Ubuntu Linux](https://web.archive.org/web/20101127071238/http://developer.symbian.org/wiki/Bootstrapping_the_Symbian_build_tools_on_Ubuntu_Linux)

3. [How to build Symbian QEMU on Ubuntu Linux](https://web.archive.org/web/20101127071350/http://developer.symbian.org/wiki/Kernel:How_to_build_Symbian_QEMU_on_Ubuntu_Linux)