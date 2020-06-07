Following config files specify a configuration layout that allows a user to put multiple live distributions on a single usb flashdrive, or optical (cd/dvd) disk.

INSTALLATION
------------
0. You will need :
- a linux system
- syslinux (preferably 4.xx or newer) installed on your linux system, along with mtools.
- a usb flash drive or a blank cd/dvd disk
- iso files of at least one supported distribution


1. Create a partition on your flashdrive, mark it bootable.
2. If you want to use Hiren's boot cd, format it to fat32, otherwise you can go with ext2/ext3.
3. Install syslinux to the partition

syslinux -i /dev/xxx  (when using fat32)

OR 

extlinux -i /mnt/directory ( When using ext2/ext3 , you need to mount your device 
and pass the mount point as parameter for extlinux command).

If you did it right, a file "ldlinux.sys" should appear in the root of the partition on usb stick.

4. Copy over the syslinux.cfg and splash.png to the root of device.

4a. Alternatively to 4. , you can instead copy files to /syslinux on the root of the drive, to reduce the clutter.

5. Copy over all .c32 files from your syslinux directory to the root of usb drive. 
Depending on the distribution they can be found in /usr/share/syslinux or /usr/lib/syslinux.

5a. Alternatively to 5. , you can opt to copy *.c32 files into /syslinux to make root directory appear cleaner.

6. Copy .cfg files for distributions you need in the same place as files in previous steps. 

7. Edit syslinux.cfg and remove/add entries to leave only distributions you will need.
7a. Alternatively you can copy over and run genmenu.sh script. It will create a new syslinux.cfg referencing existing .cfg files.

./genmenu.sh > new.cfg
mv new.cfg syslinux.cfg

8. Sometimes syslinux/extlinux won't actually install the MBR to the device ( despite what -i command is supposed to do). 
You might have to issue following command as root : 

dd if=mbr.bin of=/dev/xxx  (make sure it's not mounted, just in case), where /dev/xxx is your whole usb device (not the partition).

e.g. dd if=mbr.bin of=/dev/sdc

Usually mbr.bin will suffice, but there are other choices for nonstandard setups or GPT layouts. They will be described later on.
The mbr.bin file can be found in the same directory *.c32 syslinux files are on your system.

8. You can now try booting from the device, either on real system or by using qemu. 
If a syslinux menu shows up, you can start copying over distributions to usb stick. Refer to the next paragraph for instructions.
 
If you are experiencing problems with booting off usb stick, refer to troubleshooting section. Or you could try preparing the device from beginning.


HOW DO I PUT MY LIVE DISTRIBUTION FILES ON THE USB
--------------------------------------------------
First, you need to access the contents of the iso image of your live distribution of choice. In most cases, p7zip will do just fine.

You can extract the iso by running "7z x file.iso", or by mounting the iso image and copying out the files. 

You can also copy out files out of the iso with mc. But that's not very reliable. It often copies files out as empty, or doesn't display hidden files at all.

Once you extract the contents of the iso image, check the .cfg file for that distribution for further instructions - usually you don't need to copy over everything to the usb. For example, for Fedora read fedora.cfg file.

WHAT'S THAT UUID ? 
------------------
You might notice that some configuration files contain a phrase UUID=SOMEUUID . You will need to fix that manually.

UUID is a unique identifier of a given partition. Even in a setup with lots of disks, it can be used to locate the correct partition quickly.

First issue a command 'blkid' with your usb device plugged in to your computer to see uuids assigned to block devices on your system. 

You should get something like this

(.....)
/dev/sdc1: UUID="D88E-7075" TYPE="vfat"
(.....)

Where /dev/sdc is the usb device. So the UUID of its partition is D88E-7075 . Replace all occurences of SOMEUUID with that value in config files.

Non-fat32 filesystems will have more complex UUID's. Don't worry about that, and proceed normally.

Example : 
/dev/sda2: LABEL="boot" UUID="e27288df-82eb-44b3-90d3-32ffd6f8c26b" TYPE="ext3"

NOTE : UUID does not apply to CD media. Use label parameter instead, if possible.

WHAT'S THAT LABEL ?
-------------------
some configs specify a SOMELABEL parameter. Replace it with a label of your partition on usb. As before, use the blkid command. 

/dev/sdc1: UUID="D88E-7075" TYPE="vfat" LABEL="MYSUPERUSBPEN"

If the LABEL="...." fragment does not appear, a partition does not have a label. In this case, you can assign it one (or change existing one), by doing : 

tune2fs -L somelabel /dev/sdc1  (ext2/ext3)
dosfslabel /dev/sdc1 somelabel  (fat32)

Make sure the label is fairly unique, so that you avoid the situation where there are multiple partitions with the same label as your usb stick. That could cause boot problems.


IT DOES NOT BOOT AT ALL!
------------------------
If you have a fairly recent computer, check your BIOS for option called "USB Storage Emulation Type". It's usually set to "Auto" by default, and this setting will cause "Boot error" message on startup.  Set this to "Fixed" or "All Fixed" (depending on what's available). 


You can try booting media with qemu, by running : 

sudo qemu -hda /dev/sdc 

Where /dev/sdc is the usb device. If it boots, it means that syslinux installation is correct, but there might be a problem with your BIOS or partition layout.

Qemu treats device as classic hard disk. That means that it won't expose any issues related to USB booting, it will simply test if your bootloader installation is correct.

* The device must not be mounted when testing with qemu ! Otherwise weird things can happen !*

If it doesn't boot under qemu at all, you messed something up. Try reinstalling syslinux/extlinux MBR on the device. If you are using classic partition layout (not GPT), partition should be marked bootable as well.

Other things to try :
---------------------
- turn your usb device into fake USB-ZIP device.
	It's possible to setup <=8GB flash to appear as USB-ZIP drive. This is said to greatly improve compatibility with certain BIOS configurations, however it's recommended for =<4GB devices, somewhat risky for 8GB devices, and probably impossible for >8GB devices.

More info on it is here : http://www.knoppix.net/wiki/Bootable_USB_Key#Large_USB_keys . It basically uses mkdiskimage command from syslinux package.

- zero first 1MB of the drive before installing anything on it. You will have to redo the installation, as it will erase the MBR and probably the partition table.
dd if=/dev/zero bs=1M count=1 of=/dev/sdc 

You can try erasing more than 1MB, but it's not necessary.

- syslinux and extlinux has -s parameter that installs slower, but more compatible MBR to the device.

- There are alternate MBR bootloaders provided with syslinux that you can use. 
In syslinux directory on your system (mentioned earlier) there will be files mbr.bin , mbr_c.bin and mbr_f.bin.

The _c variant will set the drive bios is booting from as the first harddrive when ctrl is held during bootup, the _f will always do this. 
This is helpful if somehow BIOS messes up the drive order and syslinux gets confused.

- even more alternative mbr's : 
altmbr.bin will ignore the active (boot) flag on a partition and instead use a specified partition to boot. 

It needs to be configured by adding a byte with partition number to the end of the file, and then installing it into MBR. 
e.g 
echo -ne "\x03" >> altmbr.bin  	#to make it use partition #3
cat altmbr.bin > /dev/sda 	#install the MBR

altmbr*.bin file should be 440 bytes long after modification. If it's not, you did something wrong.

It has similar _c and _f variants as mentioned before.

- the gptmbr.bin is for GPT partition layouts. It also has similar _c and _f variants for troubleshooting, which work in similar way to their classic mbr counterparts.

- configure partition type as FAT16, format it with any syslinux supported filesystem. 
This helps with some kingston data traveller II 8GB flashdrives, and odd bioses. Try this if BIOS takes too long to recognize a device at boot, which makes it not appear in boot menu.

- if all else fails, you can use grub2 or other bootloader (plopbt or something else). Grub2 doesn't have some advantages of syslinux, and requires rewriting config files. It's sometimes more compatible on certain hardware configurations (e.g. older intel motherboards). You could try to chainload syslinux from grub2, that could work (untested). It's possible to do the reverse thing (hiren menu has such an option).

No USB boot? No problem.
------------------------
If your computer cannot boot from usb at all, you could make a livecd/dvd instead. 
Prepare the directory as usual, read the contents of attached buildcd.sh script for instructions. Make sure your configs do not use UUID parameter. Testing the generated ISO in qemu should provide realistic results.

Some configs will require modifications (e.g puppy linux - pmedia=cd parameter might be necessary, alpine linux might require change of parameters to load iso9660 modules, etc. ).
