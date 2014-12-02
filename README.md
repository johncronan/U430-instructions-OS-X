OS X 10.10.x installation on Lenovo U430
=======================================

Known issues: 1. SD card reader not currently supported. 2. BIOS whitelist prevents replacement of Wifi/Bluetooth card (must use USB). A BIOS hack is possible, but so far it requires physically replacing the chip on the mainboard, which is tricky. The whitelist can be circumvented by rebranding; see the forum thread for details. 3. Headphone port has distorted audio. VoodooHDA is another option, and apparently it can be made to work well.

Note: Refer to the new [mini guide]. These instructions are still here in case they provide useful info on certain details of the process.

1. In Windows, create a recovery drive (copying the recovery partition) and choose `delete the recovery partition` at the end. I needed a 16GB USB flash drive for this (which I just made a disk image of afterwards, and stored away).
1. In Disk Management, delete the `New Volume` just created and the empty `LENOVO` primary partition.
1. Resize NTFS partition according to [resize instructions].
1. (Note: the following three steps are now optional. If you skip them then you should also skip the first two lines refering to `tables.tgz` below.) Create a Linux live USB using `unetbootin` and, e.g., [Lubuntu 14.04]
1. Boot into Linux and open LXterminal:

	```
	sudo cp -pR /sys/firmware/acpi/tables tables
	sudo tar czf tables.tgz tables
	```

1. Copy the `tables.tgz` just created to `whatever/tables.tgz` on your existing Mac.
1. On your current Mac: Make sure you have Xcode and its command line tools installed. Also, change your Xcode preferences in Locations -> Advanced -> Custom, Relative to Workspace. 
1. In Terminal:

	```
	cd whatever
	git clone https://github.com/RehabMan/Laptop-DSDT-Patch.git laptop.git
	git clone https://github.com/RehabMan/Lenovo-U430-Touch-DSDT-Patch.git u430.git
	```

1. Now take a look at the README.md file in `whatever/u430.git`. Install `patchmatic` and `iasl` to your `/usr/local/bin` using the provided links. Or, try out the `download.sh` and `install_downloads.sh` scripts in the `u430.git` directory. Continue in Terminal:

	```
	tar zxf tables.tgz
	mv tables/* u430.git/native_linux/* && rmdir tables
	cd u430.git
	./disassemble.sh
	make patch
	make
	cp config.plist build/DSDT.aml ..
	cp build/ssdt4.aml ../ssdt-4.aml
	cp build/ssdt6.aml ../ssdt-6.aml
	```

1. Make sure that you have the `Install OS X Yosemite` app on your Mac, and that you've updated it to the latest version (by finding it in App Store and clicking `Download`) if it is old.
1. Follow [Booting the OS X installer on laptops with Clover UEFI] through "createinstallmedia method." At "choosing a config.plist" you should use the config.plist file in your `whatever` dir.  You can skip the USB and Ethernet kexts.
1. Copy the DSDT and the two SDST .aml files to `EFI/CLOVER/ACPI/patched/` (you could skip this, but note that later on we assume these files are located on the installer's ESP).
1. Now, on your U430: Go into the BIOS (shut down, then press the tiny button on the side) and disable `Secure Boot` under `Security`. Make sure the date is accurate.
1. Shut down again, and boot from the installer USB (tiny button, `Boot Menu`, `EFI USB Device`), selecting the first option, `install_osx`.
1. Go into `Disk Utility` and select your hard drive. On the `Partition` tab, click the `+` button and name your new partition with format "Mac OS Extended (Journaled)." Click `Apply`. (Note: if Disk Utility gets stuck, then you will have to first [create the partition] using `gpt add ...` in the Terminal, then format it with Disk Utility.)
1. Close Disk Utility and `Install` onto your newly created partition. It may stall for a long while at the end; be patient.
1. When it reboots, you should again boot off the USB installer and choose `install_osx`. The installation will complete and the system restarts again.
1. Boot off the USB one last time and choose the partition that you installed Yosemite onto. Complete the setup (skip the network step).
1. Mount your system EFI partition (`diskutil mount /dev/disk0s2`) and rename it in Finder to `EFI`. It will already be FAT32.
1. Copy your Clover package and [Clover Configurator] onto your new Yosemite install. Also copy [ssdtPRGen.sh] and the `iasl` that you got earlier.
1. Install Clover using the settings described in the previously-linked installation guide, in "Installing Clover UEFI to USB" for "HDD install."
1. Now plug in your installer USB drive so we can copy over some files (`disk1s1` should correspond to the EFI partition on the installer):

	```
	diskutil mount /dev/disk1s1
	cd /Volumes/EFI\ 1/
	cp EFI/CLOVER/config.plist /Volumes/EFI/EFI/CLOVER/
	cp EFI/CLOVER/ACPI/patched/*.aml /Volumes/EFI/EFI/CLOVER/ACPI/patched/
	cp EFI/CLOVER/drivers64UEFI/HFSPlus.efi /Volumes/EFI/EFI/CLOVER/drivers64UEFI/
	cp -R EFI/CLOVER/kexts/Other/* /Volumes/EFI/EFI/CLOVER/kexts/Other/
	rmdir /Volumes/EFI/EFI/CLOVER/kexts/10.*
	```

1. Run `Clover Configurator`, let it mount your EFI partition, then go to the `SMBIOS` tab, click the wand icon, select `MacBookAir6,2` from the drop down. You can then save your config.plist.
1. Then in Terminal:

	```
	cd files_loc
	sudo cp iasl /usr/local/bin/
	sudo ./ssdtPRGen.sh
	(say 'n' twice)
	mv ~/Desktop/ssdt.aml /Volumes/EFI/EFI/CLOVER/ACPI/patched/
	sudo pmset -a hibernatemode 0
	```

1. Now reboot. The remainder of these instructions are for historical reference, as at this point you can just run `download.sh` in the `u430.git` directory, transfer this folder over, and then use `install_downloads.sh`.
1. Finally, we have some kexts to install. Refer to RehabMan's [kext list here].  We've taken care of FakeSMC and VoodooPS2Controller already. And see the next step for info about the AppleHDA patch. For the others,  follow the link in the `Downloads` section of each README, or build from source as follows:

	```
	git clone address_from_github_page destdir
	cd destdir
	xcodebuild -alltargets
	cd ..
	```

1. The targets will be in a directory called `Release`, or placed inside each of the gits, in the `build` directory, if you built from source. For `VoodooPS2Controller` you also need to install [the daemon] and you may copy the preference pane into `/System/Library/PreferencePanes`.  As for the AppleHDA patch, the procedure is a little different. Do the `git clone`, and on the U430 you will take the kext from the same directory after the command:

	```
	./patch_hda.sh
	```

1. And use your preferred method to install all the kexts you built: GenericUSBXHCI, ACPIBatteryManager and ACPIBacklight, RealtekRTL8111, CodecCommander and AppleHDA_ALC283.

That should complete your installation! Note that you need to get a boot with kernel cache for the audio to work. Please send any feedback to <kyle@pbx.org> and I will update the instructions as needed, thanks!

Many thanks, of course, to RehabMan for pioneering the above and for the various useful scripts and patches.

[mini guide]:http://www.tonymacx86.com/laptop-compatibility/121632-lenovo-ideapad-u430-mavericks-43.html#post928795
[resize instructions]:http://ubuntuforums.org/showthread.php?t=2087466&p=12372055#post12372055
[Lubuntu 14.04]:http://cdimage.ubuntu.com/lubuntu/releases/14.04/release/lubuntu-14.04-desktop-amd64.iso
[Booting the OS X installer on laptops with Clover UEFI]:http://www.tonymacx86.com/yosemite-laptop-support/148093-guide-booting-os-x-installer-laptops-clover-uefi.html
[create the partition]:http://apple.stackexchange.com/questions/63130/create-new-partition-in-unallocated-space-with-diskutil
[Clover Grower]:https://github.com/STLVNUB/CloverGrower
[Clover Configurator]:http://www.osx86.net/files/file/49-clover-configurator/
[ssdtPRGen.sh]:https://github.com/Piker-Alpha/ssdtPRGen.sh
[kext list here]:http://www.tonymacx86.com/laptop-compatibility/121632-lenovo-ideapad-u430-mavericks-19.html#post816396
[the daemon]:https://github.com/RehabMan/OS-X-Voodoo-PS2-Controller/wiki/How-to-Install
