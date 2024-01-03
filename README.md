# Starlink Dishy Emulator for Gen 2

This repository belongs to ``Best of the Best 12th 익스 딸끄니까⭐️`` team.
We've tried to make an emulator for starlink dishy gen2. Basically we've followed up on [quarks lab starlink tools](https://github.com/quarkslab/starlink-tools) and changed some of the codes to adjust to our environment!

## 1. Settings

- OS : Ubuntu 22.04(Vmware fusion)
- Emulated device : rev3_proto2
- Method of getting the firmware : did a eMMC chip-off from the antenna, reballed it and dumped the firmware using BGA socket

## 2. Extracting parts
Once you have dumped the firmware, it will look like this.
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/b018eb5d-5794-460a-a601-a0caf96b795a)

using [quarks lab parts-extractor](https://github.com/quarkslab/starlink-tools/tree/main/parts-extractor) you will be able to get different parts like picture below
<img width="789" alt="image" src="https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/dbe849d7-cac8-40c9-8c74-aa0d59b1499c">


### 2-1. ``bootfip`` & ``fip`` partition
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

<img width="657" alt="image" src="https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/ed803c34-a9a7-45f3-8d05-aa53aa67af9b">
<img width="659" alt="image" src="https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/8b8f755e-6247-4275-86b4-745f434f3b5d">


### 2-2. ``linux`` partition
the ``linux partition`` is protected with a special method called ``SXECC``(probably spacex error correcting code). So in order to use binwalk to get the files inside, we needed to do some works to get it working
<img width="1004" alt="image" src="https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/07935903-3f0d-4dfe-aabe-269c7f44e816">

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

## 3. Emulation
Basically all the codes and files are the same from [quarks lab emulator](https://github.com/quarkslab/starlink-tools/tree/main/emulator). So clone it from the repository first. Below are specific instructions on using the codes and different codes we used to do the emulation of the second generation dishy.

### 3-1. Firmware integration
#### 1. original.img
We need the whole file we've dumped before(userarea.bin file). Copy that file and name it ``original.img``

#### 2. rootfs folder
Copy the folder we made above and move it to this directory

In the ``rootfs/usr/bin/is_production_hardware`` file, we need to change some parts for our emulator to show log in page. Like the picture below, we've changed parts that said ``1`` to ``0`` for it to work.
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/9ef47f2c-c944-41a8-8e49-ee84861edd8f)

#### 3. linux kernel
What we need is linux kernel 5.15.55 version. You can get the source code from [linux kernel 5.15.55](https://www.linuxcompatible.org/story/linux-kernel-51555-released/). After that, using the ``build-kernel.sh`` file, you can build the kernel needed for the emulator.

### 3-2. Flattend Device Tree modification
For the emulator to work, you need to get a flattened device tree file for your version.(In our case, the hardware version was ``rev3_proto2``)

Using the dumpimage tool above, you can get the device tree file. Ours was image number 18.
![image](https://github.com/bob-doduk/Starlink-Analysis-gen2/assets/102951397/97f5d2dc-1aa7-412b-aed9-a98b95707966)
After getting the image, you will get a dtb file and  it can be decompiled using ``dtc`` tool.(you can see dtc tool usage information at [dtc documentation](https://siliconbladeconsultants.com/2022/05/26/decompiling-the-linux-device-tree-dtb/))

The way we made our own dtb file is like this.
1. Get a clean dtb file of qemu-system-aarch64
   -> this can be done by following [official documentation](https://u-boot.readthedocs.io/en/stable/develop/devicetree/dt_qemu.html)
   ``qemu-system-aarch64 -machine virt -machine dumpdtb=qemu.dtb``

2. merge it with the dtb extracted
   -> ``cat  <(dtc -I dtb qemu.dtb) <(dtc -I dtb rev3_proto2.dtb | grep -v /dts-v1/) | dtc - -o merged.dtb``

After that, we did some trials to get the dtb working. You will have to add or erase parts to get it working. File uploaded is the one that fitted our environment.(I erased the dm-verity key part so you will have to find it in your own dts file. But it turns out that without the security parts, it appears to be working well)

### 3-3. emulate.sh file modification
First of all, you will have to change the ``sx-serialnum`` at ``bootargs``. You can find your own num at the ``nt-fw.bin``. This was the file extracted at ``fip_a.0`` using fiptool above.

Second, you will have to change the ``-dtb`` option file to the one you made.

Lastly, erase the ``-virtfs local,path=./disks/dish-cfg,mount_tag=dishcfg,security_model=mapped-xattr \`` option because we don't need it.

And that is it! you can now run your emulator to work. The log in id and password is ``root`` and ``falcon``.
