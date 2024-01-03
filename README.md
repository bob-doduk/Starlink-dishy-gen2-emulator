# Starlink Dishy Emulator for Gen 2

This repository belongs to ``Best of the Best 12th 익스 딸끄니까⭐️`` team.
We've tried to make an emulator for starlink dishy gen2. Basically we've followed up on [quarks lab starlink tools](https://github.com/quarkslab/starlink-tools) and changed some of the codes to adjust to our environment!

## 0. Settings

- OS : Ubuntu 22.04(Vmware fusion)
- Emulated device : rev3_proto2
- Method of getting the firmware : did a eMMC chip-off from the antenna, reballed it and dumped the firmware using BGA socket

## 1. Extracting parts
Once you have dumped the firmware, it will look like this.
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/b018eb5d-5794-460a-a601-a0caf96b795a)

using [quarks lab parts-extractor](https://github.com/quarkslab/starlink-tools/tree/main/parts-extractor) you will be able to get different parts like picture below
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/76e9a487-64c9-4a7d-ab82-85a36d0c9a3a)

### 1-1. ``bootfip`` & ``fip`` partition
First of all, you have to build ``fiptool`` to analyze these two partitions.

#### 1. building fiptool
ref: [fiptool build documentation](https://trustedfirmware-a.readthedocs.io/en/latest/getting_started/tools-build.html)
ref: [build option documentation](https://trustedfirmware-a.readthedocs.io/en/latest/getting_started/build-options.html#build-options)

To use fiptool, you need to have ``openssl`` installed on your environment. 
```
//dependency installation
sudo apt-get install -y build-essential make zlib1g-dev

//download source code
cd /usr/local/src
sudo wget https://www.openssl.org/source/openssl-3.0.11.tar.gz
sudo tar xzf openssl-3.0.11.tar.gz
cd openssl-3.0.11

//openssl compile & installation
sudo ./config --prefix=/usr/local/openssl --openssldir=/usr/local/openssl shared zlib
sudo make -j $(($(nproc) + 1))
sudo make install
export LD_LIBRARY_PATH=/usr/local/openssl/lib:$LD_LIBRARY_PATH
```

after you've installed openssl, it's time to build fiptool.

```
//ARM-TF clone
git clone https://github.com/ARM-software/arm-trusted-firmware.git

//build fiptool
cd arm-trusted-firmware/tools/fiptool
export LD_LIBRARY_PATH=/usr/local/lib64
make OPENSSL_DIR=/usr/local/src/openssl-3.0.11
```

the result page will look like this.
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/46b7fa6e-834f-4ea7-b9ab-90ead171496c)

#### 2. using fiptool
As for me, I've made a ``source`` directory under the fiptool directory and copied ``bootfip0`` & ``fip_a.0`` partitions to the source directory.

After that, unpacked each partitions using the command below.
```
//bootfip0
./fiptool unpack ./source/bootfip0 --out ./source/bootfip

//fip_a.o
./fiptool unpack ./source/fip_a.0 --out ./source/fip
```

As a result you can get files like below.
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/8485f26c-7784-40e9-b64d-f65a2a8244cf)
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/a6bceac9-07f0-432b-af19-8e58e41dfe80)

### 1-2. ``linux`` partition
the ``linux partition`` is protected with a special method called ``SXECC``(probably spacex error correcting code). So in order to use binwalk to get the files inside, we needed to do some works to get it working
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/cda94538-51ab-4b6d-87af-b48c689dde9a)

We've used codes from [quarks lab un-ecc code](https://github.com/quarkslab/starlink-tools/tree/main/unecc)
```
//code installation
git clone https://github.com/quarkslab/starlink-tools.git

//argparser installation
sudo apt-get update -y
sudo apt-get install -y python-argparse

//un-ecc
cd starlink-tools/unecc
python3 unecc.py ./source/linux_a ./source/linux
```

After that, we've used ``dumpimage tool`` to get the files we want for our emulator

#### 1. dumpimage tool
```
//u-boot installation
git clone https://github.com/SpaceExplorationTechnologies/u-boot.git

//build dumpimage tool
cd u-boot
sudo apt-get install bison flex -y
make tools-only_defconfig
make tools-only
export PATH=$PATH:$PWD/tools  # use this command at u-boot directory
```

if error like this happens,
<img width="676" alt="image" src="https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/3d754d2f-00c4-4e1f-94bf-3fc16f48a7c7">

use these two commands for trouble shooting
```
sudo apt-get install libsdl2-dev
sudo apt-get install libssl-dev
```

using this tool we can now dump the ``linux partition``
```
dumpimage -l ./source/linux
```
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/745a9792-223d-44ea-8af9-784069c36c6f)
As you can see, there are many images and configuration files included in this partition. What we need for our emulation is ``ramdisk``, ``runtime``, and ``kernel``. In our case, the image number for ramdisk was 27 and kernel was 0.
so using the command below we extracted the files we wanted
ref: [dumpimage usage documentation](https://www.gibbard.me/linux_fit_images/)
```
//ramdisk
dumpimage -T flat_dt -p 27 -o ./source/ramdisk/ramdisk.lzma ./source/linux
cd source/ramdisk
unxz ramdisk.lzma

//kernel
dumpimage -T flat_dt -p 0 -o ./source/kernel/kernel.lzma ./source/linux
cd source/kernel
unxz kernel.lzma
```

After that we used ``binwalk -Mre ramdisk`` to get the rootfs(file named ``cpio-root`` is the rootfs that we need. So I changed the name after using binwalk)

A little problem here is that the ``rootfs/sx/local/runtime`` folder is empty. Actually you can find this folder in the partition extracted folder. Partition named ``sx_a`` and ``sx_b`` has this folders so we used binwalk to get the files we wanted. 
```
binwalk -Mre sx_a
```

The ``_1080.extracted`` folder is the ``runtime`` folder that we want. So we changed its name to ``runtime`` and moved it to ``rootfs/sx/local/runtime`` folder and finally got the whole rootfs filesystem in our hand.

