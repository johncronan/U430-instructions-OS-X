OS X 10.9.x installation on Lenovo U430
=======================================

Known issues: 1. SD card reader not currently supported. 2. BIOS whitelist prevents replacement of Wifi/Bluetooth card (must use USB). A BIOS hack may be possible, if a complete copy of the current BIOS can be read or a BIOS update is released.

1. In Windows, create a recovery drive (copying the recovery partition) and choose `delete the recovery partition` at the end. I needed a 16GB USB flash drive for this (which I just made a disk image of afterwards, and stored away).
1. In Disk Management, delete the `New Volume` just created and the empty `LENOVO` primary partition.
1. Resize NTFS partition according to [resize instructions].
1. Create a Linux live USB using `unetbootin` and, e.g., [Lubuntu 14.04]
1. Boot into Linux and open LXterminal:

	```
	sudo cp -pR /sys/firmware/acpi/tables tables
	sudo tar czf tables.tgz tables
	```

1. Copy the `tables.tgz` just created to `whatever/tables.tgz` on your existing Mac.
1. On your current Mac: Make sure you have Xcode and its command line tools installed. Also, change your Xcode preferences in Locations -> Advanced -> Custom, Relative to Workspace. 
1. Grab `MaciASL_ML.zip` from [MaciASL] into your `Downloads` directory.
1. In Terminal:

	```
	cd whatever
	curl http://www.tonymacx86.com/attachments/laptop-compatibility/75686d1386006623-would-my-dell-inspiron-17-7000-hackintosh-able-iasl.zip > iasl.zip
	unzip iasl.zip
	sudo mv iasl /usr/bin/
	git clone https://github.com/RehabMan/OS-X-MaciASL-patchmatic.git patchmatic.git
	unzip ~/Downloads/MaciASL_ML.zip
	cp MaciASL.app/Contents/MacOS/iasl* patchmatic.git/
	cd patchmatic.git
	sed -e 's/macosx10.7/macosx10.8/' -i \~ MaciASL.xcodeproj/project.pbxproj
	xcodebuild -alltargets
	sudo cp build/Release/patchmatic /usr/local/bin/
	git clone https://github.com/RehabMan/Laptop-DSDT-Patch.git laptop.git
	git clone https://github.com/RehabMan/Lenovo-U430-Touch-DSDT-Patch.git u430.git
	tar zxf tables.tgz
	mv tables u430.git/linux_native
	cd u430.git
	./disassemble.sh
	make patch
	make
	cp config.plist build/dsdt.aml ..
	cp build/ssdt4.aml ../ssdt-4.aml
	cp build/ssdt6.aml ../ssdt-6.aml
	```

1. Edit the config.plist file in `whatever` dir, and add the following to the SMBIOS key, right after `<dict>` line:

	```
	                <key>ProductName</key>
	                <string>MacBookAir6,2</string>
	                <key>ProductFamily</key>
	                <string>MacBookAir</string>
	```

1. Make sure that you have the `Install OS X Mavericks` app on your Mac, and that you've updated it to the latest version (by finding it in App Store and clicking `Download`) if it is old.
1. Prepare an updated Clover installer package using [Clover Grower]. To use Clover Grower you just clone that git repo and run `CloverGrower.command`. You need Clover r2678 or later for the installer to detect `/dev/disk0s2` as the ESP (needed later).
1. Follow [Install OS X Mavericks using Clover] for `UEFI Boot Mode` through step 2. At the end of step 2 you should use the config.plist file in your `whatever` dir. You can skip the Ethernet kext (and NullCPUPowerManagement if you add the DSDT and SDST .aml files to `EFI/CLOVER/ACPI/patched/`).
1. Now, on your U430: Go into the BIOS (shut down, then press the tiny button on the side) and disable `Secure Boot` under `Security`.
1. Shut down again, plug in a USB mouse and keyboard, and boot from the installer USB (tiny button, `Boot Menu`, `EFI USB Device`), selecting the first option, `Install OS X Mavericks`.
1. Go into `Disk Utility` and select your hard drive. On the `Partition` tab, click the `+` button and name your new partition with format "Mac OS Extended (Journaled)." Click `Apply`. (Note: if Disk Utility gets stuck, then you will have to first [create the partition] using `gpt add ...` in the Terminal, then format it with Disk Utility.)
1. Close Disk Utility and `Install` onto your newly created partition. It may stall for a long while at the end; be patient.
1. When it reboots, you should again boot off the USB installer and choose `Install OS X Mavericks`. The installation will complete and the system restarts again.
1. Boot off the USB one last time and choose the partition that you installed Mavericks onto. Complete the setup (skip the network step).
1. Mount your system EFI partition (`diskutil mount /dev/disk0s2`) and rename it in Finder to `EFI`. It will already be FAT32.
1. Copy your Clover package and [Clover Configurator] onto your new Mavericks install. Also copy [ssdtPRGen.sh] and the `iasl` that you got earlier.
1. Install Clover using the settings shown in the previously-linked installation guide, in Step 4 under `UEFI-capable systems`.
1. Now plug in your installer USB drive so we can copy over some files (`disk1s1` should correspond to the EFI partition on the installer):

	```
	diskutil mount /dev/disk1s1
	cd /Volumes/EFI\ 1/EFI
	cp EFI/CLOVER/config.plist /Volumes/EFI/EFI/CLOVER/
	cp EFI/CLOVER/ACPI/patched/*.aml /Volumes/EFI/EFI/CLOVER/ACPI/patched/
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

1. Now reboot. Finally, we have some kexts to install. Refer to RehabMan's [kext list here]. For each of these (except FakeSMC, which we've already taken care of) do the following, either on the existing Mac you used previously or on the U430 after installing Xcode:

	```
	git clone address_from_github_page destdir
	cd destdir
	xcodebuild -alltargets
	cd ..
	```

1. The targets will be placed inside each of these in the `build` directory. For `VoodooPS2Controller` you also need to install [the daemon] and copy the preference pane into `/System/Library/PreferencePanes`. And the procedure for the last, the AppleHDA patch, is different. Run it from your OS X on the laptop, and you will take the kext from the same directory after the command:

	```
	./patch_hda.sh
	```

1. And use your preferred method to install all the kexts you built: GenericUSBXHCI, VoodooPS2Controller, ACPIBatteryManager and ACPIBacklight, RealtekRTL8111, CodecCommander and AppleHDA_ALC283.

That should complete your installation! Note that you need to get a boot with kernel cache for the audio to work. Please send any feedback to <kyle@pbx.org> and I will update the instructions as needed, thanks!

Many thanks, of course, to RehabMan for pioneering the above and for the various useful scripts and patches.

[resize instructions]:http://ubuntuforums.org/showthread.php?t=2087466&p=12372055#post12372055
[Lubuntu 14.04]:http://cdimage.ubuntu.com/lubuntu/releases/14.04/release/lubuntu-14.04-desktop-amd64.iso
[MaciASL]:http://sourceforge.net/projects/maciasl/
[Install OS X Mavericks using Clover]:http://www.tonymacx86.com/mavericks-desktop-guides/125632-how-install-os-x-mavericks-using-clover.html
[create the partition]:http://apple.stackexchange.com/questions/63130/create-new-partition-in-unallocated-space-with-diskutil
[Clover Grower]:https://github.com/STLVNUB/CloverGrower
[Clover Configurator]:http://www.osx86.net/files/file/49-clover-configurator/
[ssdtPRGen.sh]:https://github.com/Piker-Alpha/ssdtPRGen.sh
[kext list here]:http://www.tonymacx86.com/laptop-compatibility/121632-lenovo-ideapad-u430-mavericks-19.html#post816396
[the daemon]:https://github.com/RehabMan/OS-X-Voodoo-PS2-Controller/wiki/How-to-Install
