OS X 10.9.x installation on Lenovo U430
=======================================

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
	                <key>SerialNumber</key>
	                <string>ENTERyour17digits</string>
	```

1. Make sure that you have the `Install OS X Mavericks` app on your Mac, and that you've updated it to the latest version (by finding it in App Store and clicking `Download`) if it is old.
1. Follow [Install OS X Mavericks using Clover] for `UEFI Boot Mode` through step 2. At the end of step 2 you should use the config.plist file in your `whatever` dir. You can skip the Ethernet kext (and NullCPUPowerManagement if you add the DSDT and SDST .aml files to `EFI/CLOVER/ACPI/patched/`).
1. Now, on your U430: Go into the BIOS (shut down, then press the tiny button on the side) and disable `Secure Boot` under `Security`.
1. Shut down again, plug in a USB mouse and keyboard, and boot from the installer USB (tiny button, `Boot Menu`, `EFI USB Device`), selecting the first option, `Install OS X Mavericks`.
1. Go into `Disk Utility` and select your hard drive. On the `Partition` tab, click the `+` button and name your new partition with format "Mac OS Extended (Journaled)." Click `Apply`. (Note: if Disk Utility gets stuck, then you will have to first [create the partition] using `gpt add ...` in the Terminal, then format it with Disk Utility.)
1. Close Disk Utility and `Install` onto your newly created partition. It may stall for a long while at the end; be patient.
1. When it reboots, you should again boot off the USB installer and choose `Install OS X Mavericks`. The installation will complete and the system restarts again.
1. Boot off the USB one last time and choose the partition that you installed Mavericks onto. Complete the setup (skip the network step).
1. Mount your system EFI partition (`diskutil mount /dev/disk0s2`) and rename it in Finder to `EFI`. It will already be FAT32.
1. Download a [Clover snapshot] and copy it onto your new Mavericks install. Also copy over an [fdisk440 binary] and the [Ethernet kext]. Then in Terminal:

	```
	chmod 755 fdisk440
	sudo cp fdisk440 /usr/bin/
	cd cloverefiboot-code-????/CloverPackage/CloverV2/BootSectors
	sudo fdisk440 -f boot0.bin -u -y /dev/rdisk0
	```

1. And, assuming that the EFI partition is `disk0s2`:

	```
	sudo dd if=/dev/rdisk0s2 count=1 bs=512 of=origbs
	cp boot1f32alt newbs
	dd if=origbs of=newbs skip=3 seek=3 bs=1 count=87 conv=notrunc
	sudo dd if=newbs of=/dev/rdisk0s2 count=1 bs=512
	```

1. Mount the installer USB drive's EFI partition also with `diskutil mount /dev/disk1s1`. Then,

	```
	cd /Volumes/EFI/EFI/Boot
	mv bootx64.efi bootx64orig.efi
	cp /Volumes/EFI\ 1/EFI/BOOT/BOOTX64.efi bootx64.efi
	cd ..
	cp -R /Volumes/EFI\ 1/EFI/CLOVER .
	rm -rf drivers64UEFI/Emu* drivers64UEFI/Part*
	cp -R somewhere/RealtekRTL81xx.kext kexts/10.9/
	cd
	diskutil unmount /dev/disk1s1
	diskutil unmount /dev/disk0s2
	sudo pmset -a hibernatemode 0
	```


[resize instructions]:http://ubuntuforums.org/showthread.php?t=2087466&p=12372055#post12372055
[Lubuntu 14.04]:http://cdimage.ubuntu.com/lubuntu/releases/14.04/release/lubuntu-14.04-desktop-amd64.iso
[MaciASL]:http://sourceforge.net/projects/maciasl/
[Install OS X Mavericks using Clover]:http://www.tonymacx86.com/mavericks-desktop-guides/125632-how-install-os-x-mavericks-using-clover.html
[create the partition]:http://apple.stackexchange.com/questions/63130/create-new-partition-in-unallocated-space-with-diskutil
[Clover snapshot]:http://sourceforge.net/p/cloverefiboot/code/HEAD/tree/
[fdisk440 binary]:https://github.com/philippetev/HP-ProBook-Installer/blob/master/Chameleon/usr/sbin/fdisk440
[Ethernet kext]:http://www.tonymacx86.com/downloads.php?do=file&id=216
