# Bondmachine_zedboard_buildroot

## How to make a bondmachine accelerator on Zynq ZedBoard.

The following guide is a step by step tutorial on how to create a bondmachine accelerator using the Zynq ZedBoard.</br>
This tutorial can be broken down into three blocks:
in the first one is shown how to create the block design using Vivado and how to export the bitstream and the programmable logic; the second block will focus on Linux-based image creation using buildroot; finally in the last block is shown how to test the accelerator directly from the OS.
Requirements:
<ul>
<li> Board ZedBoard </li>
<li> USB to serial converter </li>
<li> Vivado 2020.2 with Vitis </li>
<li> Buildroot 2020.2 </li>
<li> Bondmachine sources </li>
</ul>


## Create block design

Create a new project (RTL project) on Vivado and be sure to select the correct part and product family of the ZedBoard (xc7z020clg484-1) or select ZedBoard in boards tab</br>

Open project manager (usually open by default), and firstly "Create Block Design" after click on "Add IP" and add a Zynq Processor:
</br>
<img src="media/images/add-ip.png" alt="drawing" width="300"/>
</br>
Now run a "Block Automation"
<br/>
<img src="media/images/8-design.png" alt="drawing" width="600"/>
<br/>

Now it's time to create the bondmachine ip module. The bondmachine module is essentialy a simple processor which increments a 8 bit value that is passed through the correct AXI Memory Mapped address. To do that, go to *tools* -> *Create and package new IP* -> *Create a new AXI4 Peripheral*. Call the module **bondmachine** as the name for the new IP. Leave all the other settings as they are by default. In the *Create Peripheral* menù select *Edit Ip*. </br>
The ip package manager will open and here you have to remove the files generated by default from Vivado and add the three files inside this repository as shown in the figures below.
</br>
<img src="media/images/5-edit-bm-ip-1.png" alt="drawing" width="500"/>
</br>
Now you can importa using the Add Sources the three Bond machine files in bondmachine-src:
<br>
<img src="media/images/6-edit-ip-bm-2.png" alt="drawing" width="500"/>
<br>
Go to *Review and package* and click on *re-package IP*. Now come back to Vivado and add the new IP module just created:
<br>
<img src="media/images/add-bm-ip.png" alt="drawing" width="500"/>
<br>
Finally, select *run block automation*. The result should look like the the following:
<br>
<img src="media/images/4-final-block-diagram.png" alt="drawing" width="600"/>
<br>
Now, validate the design by clicking the button *validate design* .
Create the vhdl code by selecting the block design, right-click and *create HDL Wrapper* .
<br>
<img src="media/images/hdl-wrapper.png" alt="drawing" width="600"/>
<br>

Now you are ready to generate the bitstream! Before do that, go to *tools* -> *settings* -> *bitstream* and check the box *bin_file* in order to generate also the *.bin* file.</br>
Click on *run synthesis* and wait until the entire process ends. 
After that, click on *run implementation* and at the click on *generate bitstream*. The bistream file will be generate after this last process and you will find it inside your project directory (<project_name>.runs/impl_1/<bitstreamfile.bin>).

<br>
<img src="media/images/las-bistream.png" alt="drawing" width="600"/>
<br>

Before closing vivado there is one last thing left, export the hardware! Go to *File* -> *Export* -> *Export hardware* and select *Include bitstream*. The generated file has an .xsa extension and will be located in the root directory of your project.
In the root directory of your project, create a folder and call it *DTS*: it will be useful to you when exporting dts files.
Now you can close Vivado and free up a lot of RAM!
<br>
<br>
It's time to generate all the DTS's file necessary to generate the device tree blob file. This file is "compiled" by a special compiler which produces the correctly binary that can be interpreted by U-boot and Linux. </br>
Open a CLI and clone the xilinx device tree repository inside the DTS dir.
````
git clone https://github.com/Xilinx/device-tree-xlnx
cd device-tree-xlnx
git checkout xilinx-v20XX.X
````

Note: *xilinx-v20XX.X* need to specify the vivado version yoou are using, in our case should be:
````
git checkout xilinx-v2020.2
````
and source Xilinx Tools (source /path/to/tools/Vivado/2020.2/settings64.sh).
Now chagedir into the root of the Vivado project and
Open the xsct console by typing *xsct* and then type the following commands:
````
hsi open_hw_design design_name.xsa 
hsi set_repo_path /path/to/device/tree_xlnx_repository 
hsi create_sw_design device-tree -os device_tree -proc ps7_cortexa9_0
hsi generate_target -dir /path/to/dts/folder/prev/created 
hsi close_hw_design design_name
exit 
````

go to the DTS directory of the project and check inside the dts folder if all the newly generated files are present. There should be the following files:

<ul>
<li> device-tree.mss  </li>
<li> include (directory) </li>
<li> pcw.dtsi  </li>
<li> pl.dtsi  </li>
<li> skeleton.dtsi  </li>
<li> system.dts  </li>
<li> system-top.dts  </li>
<li> zynq-7000.dtsi </li>
</ul>

Generate a complete *.dts* file using the **gcc** compiler with the following command: </br>
````
gcc -I dts -E -nostdinc -undef -DDTS -x assembler-with-cpp -o full-system.dts system-top.dts
````
And finally create the *.dtb* file using the **device tree compiler** tool:</br>
````
dtc -O dtb -o zedboard-bm.dtb full-system.dts
````
Well done! Keep aside the two keys files produced by all these steps: the bitstream file and the device tree blob file because they will be useful and essential for the functioning of the system.

## Create the Linux-based image with buildroot

Get the *2020.11.4* version of buildroot and navigate inside the main folder. 
For the creation of the image you will use the zynq configuration file inside the *configs* folder of this repository. In order to do that, type the following command:
````
make zynq_zed_defconfig
````
and now type 
````
make 2>&1 | tee build.log
````
to perform a first build (this first build can avoided). This phase takes a few minutes, take a coffee, even two.</br>
Now we can configure and prepare a proper SD images. In the buildroot dir:
````
make menuconfig
````
Now go to: "Kernel --> In-tree Device Tree Source file names" and change change the name to "zedboard-bm" (or in any case change the value to the filename of the  dtb file in the DTS dir). Now we want to enable OpenSSH server thus: "Target packages  --->  Networking applications  ---> enable OpenSSH". Once done save and exit from the menuconfig, and:
````
make linux-menuconfig
````
go to "CPU Power Management" and disable   "CPU Frequency scaling disable", again save and exit.
<br>
Now we are ready to build again the sdcard thus: 
````
make 
````
The make will stop with an error showing where you need to place the custom dtb file, in my case:
````
cp /home/redo/buildrootzedboard/DTS/zedboard-bm.dtb ./output/build/linux-4.16/arch/arm/boot/dts/zedboard-bm.dtb
make 
````

At the end of the process, the images is ready but we need to copy there the bitsteam: 

````
cd output/images/
sudo kpartx -a sdcard.img
sudo mount /dev/mapper/loop27p1 /mnt/
cd /mnt
sudo cp /home/redo/buildrootzedboard/buildrootzedboard.runs/impl_1/design_1_wrapper.bin  ./
cd /pub3/redo/buildroot/output/images/
sudo umount /mnt 
sudo kpartx -d ./sdcard.img
````
Note the name of the loop device could be different, and quite surely the name and 
location of the bitstream file will be different. Now we can copy the images into the sdcard:
````
sudo dd if=./sdcard.img  of=/dev/sdb 
````
Clearly you nedd to select the proper device name sdb in my case. Insert the sdcard into the zedboard
and power-on:

<br>
TODO test from here need to remove some ,,
<br>

<ul>
<li> boot.bin  </li>
<li> rootfs.cpio </li>
<li> rootfs.cpio.uboot  </li>
<li> uImage  </li>
<li> rootfs.cpio.gz  </li>
<li> rootfs.tar  </li>
<li> u-boot.img  </li>
</ul>

Take an sd-card and create two partitions, one of them bootable. Refer to this guide to know how to that: *https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842385/How+to+format+SD+card+for+SD+boot*

Copy the following files inside the boot partition:
<ul>
<li> boot.bin  </li>
<li> design.bin (from the Vivado build previously generated) </li>
<li> ebaz4205-zynq7.dtb (previously generated)  </li>
<li> rootfs.cpio.uboot  </li>
<li> u-boot.bin  </li>
<li> u-boot.img  </li>
<li> uEnv.txt  </li>
<li> uImage  </li>
</ul>

You can copy in this partition also the necessary file to interact with the bondmachine module through the AXI protocol. 
Clone *https://github.com/aimeemikaelac/xilinx_zedboard_c/blob/master/src/gpio-dev-mem-test.c* and create and executable usable on arm processors.
````
arm-linux-gcc -o gpio-dev-mem-test gpio-dev-mem-test.c
````
Copy the executable inside the boot partition.

Unzip *rootfs.tar* inside the second partition of the sd-card.
Insert the sd-card into the slot, use an ethernet cable if you need internet, use the USB to serial converter to see the output and to interact with the operating system. 

To test the accellerator you can use the executable previously created. Type
```
./gpio-dev-mem-test -g 0x43c00004 -o 1 
```
To send the number *1* to the bondmachine accelerator. 0x43c00004 is the memory address of the AXI memory mapped interface.
If the system does not crash, you can check using devmem if the number has been incremented:
```
devmem 0x43c00000
```
(the output should be 2)

### References

*http://bondmachine.fisica.unipg.it/*</br>
*https://github.com/embed-me/ebaz4205_buildroot*</br>
*https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842369/Build+Linux+for+Zynq-7000+AP+SoC+using+Buildroot*</br>
*https://xilinx-wiki.atlassian.net/wiki/display/A/Build+Device+Tree+Blob*
