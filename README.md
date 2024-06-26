# documenting-cva6
Rough notes produces while setting up the CVA design on the Nexys Video FPGA platform. See original project [here](https://github.com/openhwgroup/cva6)

# Setting CVA6 on Nexys Video (still experimental)

* Commit used for nexys_video testing: <80254abaeeeb46774e5405ca1da3ced378445bc0>
* Most information came from (most likely more up-to-date):
  - CVA6 bug https://github.com/openhwgroup/cva6/issues/2116
  - CVA6 repo [readme](https://github.com/openhwgroup/cva6?tab=readme-ov-file)
  - CVA6 toolchain repo builder [readme](https://github.com/openhwgroup/cva6/blob/master/util/gcc-toolchain-builder/README.md#Prerequisites)
  - CVA6-SDK repo [readme](https://github.com/openhwgroup/cva6-sdk)

Initial clone:

```
$ cd
$ git clone https://github.com/Saute0212/cva6.git saute0212_cva6
$ cd saute0212_cva6
$ git submodule update --init --recursive
```

(Using Saute0212's CVA6 repo, as per his request:
https://github.com/openhwgroup/cva6/pull/1925#issuecomment-2109032818)

Toolchain:

```
$ sudo apt-get install autoconf automake autotools-dev curl git libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool bc zlib1g-dev
$ cd util/gcc-toolchain-builder
$ sh get-toolchain.sh
$ export RISCV=$HOME/RISCV_nv
$ sh build-toolchain.sh $RISCV
```

Store environment vars at the end of `~/.bashrc` (or `$HOME/.bashrc`).
The terminal will need to be re-opened for the new variables to be sourced.
It is assumed that the RISC-V toolchain is going to be installed to
the `$HOME/RISCV_nv` directory.

```
export CVA6_REPO_DIR=$HOME/saute0212_cva6
export RISCV=$HOME/RISCV_nv
export RISCV_PREFIX=$RISCV/bin/riscv64-unknown-
export RISCV_GCC=$RISCV/bin/riscv64-unknown-gcc
export CV_SW_PREFIX=riscv64-unknown-elf-
export PATH=$RISCV/bin:$PATH
```

## Change compile flags for bootrom on Nexys Video

**NOTE**: This is already done in Saute0212's branch:
https://github.com/Saute0212/cva6/commit/80254abaeeeb46774e5405ca1da3ced378445bc0

Adjust the compile architecture `-march` flag for bootrom to work
on Nexys Video (see issue comment
https://github.com/openhwgroup/cva6/pull/1925#issuecomment-2100932367)

In `cva6/corev_apu/fpga/src/bootrom/Makefile`, change:
```
-march=rv64imac_zba_zbb_zbs_zbc_zicsr_zifencei
```

To:

```
-march=rv64imac_zicsr
```

This would allow to boot Linux

## For RTL Verilator verification

```
$ sudo apt-get install help2man device-tree-compiler
```

Create a virtual python env (needed for newer Linux distros where Python
packages cannot be installed system-wide):

```
$ sudo apt install python3-venv
$ python3 -m venv cva6-venv
$ source ~/git_repos/cva6-venv/bin/activate
```

From this point on, Python dependencies can be installed:

```
$ pip3 install -r verif/sim/dv/requirements.txt
```

Setting up Spike and Verilator for simulation. The smoke-tests will take a
while (depending on your computer hardware):

```
# DV_SIMULATORS is detailed in the next section
$ export DV_SIMULATORS=veri-testharness,spike
$ bash verif/regress/smoke-tests.sh
```

## To generate FPGA bitstream

```
$ export BOARD=nexys_video
$ make fpga
```


# Generating Linux images

```
$ git clone -b nexys-video-test https://github.com/amiroshni/cva6-sdk.git
$ cd cva6-sdk
$ git submodule update --init --recursive
```

Install dependencies

```
$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev libusb-1.0-0-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev device-tree-compiler pkg-config libexpat-dev
```

Unset the RISCV variable (cva6-sdk will setup its own):

```
$ unset RISCV
```

## Nexys Video change

**(NOTE: This is already done in my nexys-video-test branch which was cloned)**

- Info: <https://github.com/openhwgroup/cva6/pull/1925#issuecomment-2053688518>

Disable the ethernet controller in the configuration (doesn't work on Nexys Video yet):

```
$ sed -i 's/BR2_PACKAGE_ETHTOOL=y/BR2_PACKAGE_ETHTOOL=n/' configs/buildroot64_defconfig
$ sed -i -e 's/CONFIG_NET=y/CONFIG_NET=n/' -e 's/CONFIG_INET=y/CONFIG_INET=n/' -e 's/CONFIG_NETDEVICES=y/CONFIG_NETDEVICES=n/' -e 's/CONFIG_LOWRISC_DIGILENT_100MHZ=y/CONFIG_LOWRISC_DIGILENT_100MHZ=n/' configs/linux64_defconfig
$ make images
```

# Writing Linux image to SD card

* **NOTE**: It is strongly recommended to use a 4GB micro SD card, as 16GB
didn't work. This was also recommended by @Saute0212.

Download the URL data (I've no idea what this is, just following instructions here
https://github.com/openhwgroup/cva6/pull/1925#issuecomment-2053688518):

```
$ wget -nc -P $HOME/cva6-sdk/install64/ https://github.com/openhwgroup/cva6-sdk/releases/download/v0.3.0/bbl.bin
```

Format SD card (assumes cva6-sdk is cloned to user's home):

```
$ sudo make flash-sdcard PAYLOAD=bbl.bin SDDEVICE=/dev/[YOUR SD CARD DEVICE] RISCV=$HOME/cva6-sdk/install64
```

# Testing on FPGA

Plug in both USB connections from Nexys Video to PC (PROG and UART).
Upload bitstream to FPGA using Vivado.
Connect via UART to the FPGA using a terminal emulator (pico/nano/putty etc.).
Baud rate needs to be 57600.

## Current results - Linux working!

After switching to 4GB micro SD card, the Linux kernel was able to boot, and
we logged in as root and tried some of the standard utlities:
https://gist.github.com/shriyasharma11/8afee5a594f2f05512551a72740426cd

# Executing a 'Hello World' binary under Linux

For now a separate branch has been created for a simple 'Hello World' file.
https://github.com/amiroshni/cva6-sdk/tree/test-hello-nv

The trick is to create a partition in between the firmware payload and
linux kernel. See commit here for the changes:
https://github.com/openhwgroup/cva6-sdk/commit/edc03517a7c9f5375726fa0058ec9a4890dbc286

**NOTE:** The plan is to put an actual filesystem (ext4 or fat32) in this new
partition, but for now just as a test, a simple 'Hello World' binary is
written and read manually once the Linux kernel boot to prompt on the FPGA.

```
git checkout test-hello-nv
make hello
ls -ll HelloWorld/hello
```

The last command is needed to find out the binary size in bytes (will be
needed once inside the FPGA Linux environment).

Upload the image:

```
sudo make flash-sdcard PAYLOAD=bbl.bin SDDEVICE=/dev/[YOUR SD CARD DEVICE] RISCV=$HOME/cva6-sdk/install64
```

Once CVA6 Linux on the Nexys Video is loaded with these new SD card partitions,
copy the `hello` binary from the 2nd partition:

```
# dd if=/dev/mmcblk0p2 of=hello
# truncate -s [hello binary size]
# chmod u+x hello
# ./hello
Hello World!
```

The reason why truncation is required, is because the `hello` binary doesn't
take up an exact number of sectors. In my colleague's example, the binary
was 1072 bytes (which took up 3 512-byte sectors):
https://gist.github.com/shriyasharma11/96f44ef04281130d0ced6de718ee028a#file-putty-log-L109

Once a working filesystem is in place, the entire RISC-V toolchain can be
on the card, and real development/testing can take place.

**NOTE:** Ejecting the SD card while FPGA was operational caused a reset,
so plugging in a different SD card wasn't an option.

# (Incomplete) Setting up a middle partition on the SD card for transferring files

For now, this is a manual process (but similar to instructions in the Makefile),
and the max size of the second partition which didn't crash was 64MiB
(`262144-131072 = 131072 512-byte sectors = 67108864 bytes = 64 MiB`).

```
# sgdisk --clear -g --new=1:2048:29796 --new=2:131072:262144 --new=3:512M:0 --typecode=1:3000 --typecode=2:8300 --typecode=3:8300 /dev/[SD CARD]
# mkfs -t ext3 /dev/[SD CARD PART 1]
# mkfs -t ext3 -L fpgastorage /dev/[SD CARD PART 2]
# mkfs -t ext3 /dev/[SD CARD PART 3]
# dd if=install64/bbl.bin of=/dev/[SD CARD PART 1] status=progress oflag=sync bs=1M
# dd if=install64/uImage of=/dev/[SD CARD PART 3] status=progress oflag=sync bs=1M
# mount /dev/[SD CARD PART 2] /[PATH TO MOUNTED PARTITION]/fpgastorage/
# cp HelloWorld/hello /[PATH TO MOUNTED PARTITION]/fpgastorage/
```

When we tried a bigger partition size,
(`1015808-131072 = 884736 512-byte sectors = 452984832 bytes = 432 MiB`),
but mounting the partition in the buildroot environment caused the SD card to
reset, see the previous gist log:
https://gist.github.com/shriyasharma11/19f975f3ee7783b6012f175546ec6b87

Our plan was to copy the buildroot host toolchain located here:

```
[CVA6-SDK ROOT]/buildroot/output/host
```

But this entire toolchain is too big to fit into a 64 MiB partition
(about 200 MiB).
