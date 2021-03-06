# live-usb-creator

This project generates a LiveUSB image with CentOS 7.5.1804 and nCipher drivers/support tools for use in machines with
HSMs.

## Building image

Three dependencies need to be fetched out-of-band.

Set the following files in place in the same directory as the `Vagrantfile`.

* CodeSafe-linux64-dev-12.20.51.iso (1.9GB): supplied by the HSM vendor.
* CentOS-7-x86_64-Everything-1804.iso (8.8GB): `curl -O http://mirror.seas.harvard.edu/centos/7.5.1804/isos/x86_64/CentOS-7-x86_64-Everything-1804.iso` (or download with the
included `CentOS-7-x86_64-Everything-1804.torrent`)
* kernel-devel-3.10.0-862.2.3.el7.x86_64.rpm (16MB): `curl -O http://mirror.centos.org/centos/7.5.1804/updates/x86_64/Packages/kernel-devel-3.10.0-862.2.3.el7.x86_64.rpm`

Verify the following SHA256 sums:

```
$ shasum -a 256 CodeSafe-linux64-dev-12.20.51.iso CentOS-7-x86_64-Everything-1804.iso kernel-devel-3.10.0-862.2.3.el7.x86_64.rpm
bdd82fb64e269565c0f718b7b81b4d75dcbd66eaa9ae357f32b62cf87af7ab71  CodeSafe-linux64-dev-12.20.51.iso
99723c8b87dcec21df8aed23de1eb810346d42cfd2a3fafafe70a68296053417  CentOS-7-x86_64-Everything-1804.iso
ac64e0995558acf3874b2ccbdce8f3b8aae2b6d494256c4bf1ae093aec0f1c18  kernel-devel-3.10.0-862.2.3.el7.x86_64.rpm
```

This workflow uses the tool [Vagrant](https://www.vagrantup.com/) to orchestrate the creation and provisioning of a VirtualBox VM from a CentOS-provided base VM.

Build the image in a CentOS VM guest as follows:

```
# install virtualbox using brew cask if virtualbox is not installed
brew cask install virtualbox
# install vagrant using brew cask if vagrant is not installed
brew cask install vagrant
# install vagrant-vbguest if vagrant-vbguest is not installed
vagrant plugin install vagrant-vbguest
# remove the existing centos/7 image, if one exists
vagrant box remove -f centos/7
# install centos 7 vagrant box version 1804.02 (checksum via https://cloud.centos.org/centos/7/vagrant/x86_64/images/sha256sum.txt.asc)
vagrant box add --checksum-type sha256 --checksum 83f86355d8d1ea3001e377f67cc4a9ff15d827bea83a1878df1bcfd0a0dcab0f --provider virtualbox --box-version 1804.02 centos/7
# boot up a virtualbox VM to run the install in
vagrant up
# enter the VM
vagrant ssh
# kick off the build, from the VM (takes about 25 minutes)
/vagrant/build.sh
# will create boot.iso (1.1G) in the current directory (i.e. outside the VM) when finished
```

## Writing image to USB drive

1. Identify device path to disk (such as `/dev/disk3`) with `diskutil list external physical`
2. Unmount all volumes on that disk with `diskutil unmountDisk /dev/disk3`
3. Identify the “raw” device path by replacing `disk` with `rdisk` in the device path. (E.g. `/dev/disk3` becomes `/dev/rdisk3`.) This speeds up writes by 3–4× in very informal testing.
3. Write ISO to that “raw” device path with `sudo dd bs=1m if=boot.iso of=/dev/rdisk5`. Remember that on MacOS you can check in on progress by sending SIGINFO with Ctrl+T. (Takes about 30 minutes in one test.)
4. Eject the disk with `diskutil eject /dev/disk3`
5. Remember to hold F12 during startup to get a boot menu. You’ll then select something like `UEFI: SanDisk`.

## Build process

In general there are 7 phases of the build process, all kicked off in sequence when `livemedia-creator` is invoked in `build.sh`:

1. creation of an ext4 filesystem image (at e.g. `/tmp/build/diskLdYfQA.img`) in the `make_image` function of `/sbin/livemedia-creator`
2. installation of CentOS 7 onto that image by `anaconda`, as driven by the `rhel7-livemedia.ks` kickstart, and as triggered in `novirt_install` function of `/sbin/livemedia-creator`
3. creation of a compressed squashfs image (at e.g. `/var/tmp/tmpkd2g30/images/install.img`) from that ext4 image, as triggered in the `make_squashfs` function of `/sbin/livemedia-creator`
4. bundling of that squashfs image with some live media bootstrapping stuff by the `pylorax.TreeBuilder` as driven by the template `/usr/share/lorax/live/x86.tmpl`
5. invocation of `mkisofs` (also by `pylorax.TreeBuilder`) to slurp all of the files from the previous step into a bootable ISO (at e.g. `/var/tmp/tmpkd2g30/images/boot.iso`)
6. post-processing that ISO image with `isohybrid` to allow hybrid booting as a CD-ROM or as a hard disk.
7. embedding of an MD5 checksum by `implantisomd5` into an unused section of that ISO

### Checking build progress

One of the most time-intensive (and quiet) stages of this process is the `mksquashfs` operation (`Creating 4.0 filesystem on /var/tmp/tmp??????/images/install.img, block size 131072.`). You can get a rough check on progress by comparing `install.img` size to `install.img` size in a previous build, in a separate `vagrant ssh` session (will output some value between 0.0% and 100.0%, where 100.0% is completely done with the creation of the install image, assuming output sizes are in fact identical):

```
sudo sh -c "stat --printf='%s' /var/tmp/tmp*/images/install.img | python -c 'import sys; print \"%0.1f%%\" % (float(int(sys.stdin.read())) / 1045090304.0 * 100.0)'"
```

## Hacking on this project
Here are the useful files you may want to tweak.

VM related stuff:
* `Vagrantfile`: Provisions a VirtualBox VM in which the live system ISO will be built using the Vagrant tool.
* `bootstrap.sh`: A shell script that runs at the very end of VM bringup. (Currently just mounts the CodeSafe ISO.)
* `build.sh`: Manually run by the user to kick off the ISO build. Currently includes some patches for Lorax and Kickstart files applied before running the build proper with `livemedia-creator`.

Install related stuff:

* `rhel7-livemedia.ks`: This is the “kickstart” file that scripts the CentOS install by anaconda. Modified from the version that exists on the VM guest under the path `/usr/share/doc/lorax-*/rhel7-livemedia.ks`. I’ve tried to factor out most of the interesting stuff so you don’t have to touch this gross file. See my changes versus the CentOS/Fedora example liveiso kickstart by running `diff rhel7-livemedia.ks.orig rhel7-livemedia.ks`. Mostly I removed the GNOME desktop/X11, and added in hooks for the below.
* `additional_packages`: This file contains a newline-separated list of yum packages that will be installed onto the live ISO. Blank lines and comments starting with `#` are ignored. (Technical details: the `add_additional_packages.py` script adds these to the %packages section of the base kickstart script before running livemedia-creator.)
* `install_scripts/0_post_install_nochroot`: This bash script is executed by anaconda (the CentOS installer) directly after install (but before `install_scripts/1_post_install_chroot`). From the point of view of the script, the newly installed CentOS (soon to be implanted in our live ISO) lives at `/mnt/sysimage/opt/`, and this directory (the one containing the `Vagrantfile`) lives at `/vagrant/`. That makes this bash script the ideal place to copy files onto the root filesystem of the newly installed CentOS.
* `install_scripts/1_post_install_chroot`: This bash script is executed by anaconda (the CentOS installer) directly after install (but after `install_scripts/0_post_install_nochroot`). From the point of view of the script, the newly installed CentOS (soon to be implanted in our live ISO) is chrooted to `/`. That makes this bash script the ideal place to run Linux binaries (like useradd, systemctl, yum-config-manager, etc.) that change system state.

Pretty bootloader related stuff:
* `patches/usr/share/lorax/live/`: includes changes to SYSLINUX/ISOLINUX and GRUB2 configs.
* `otf_fonts/`: OTF fonts (now just SQ Market Bold)
* `grub_fonts/`: PF2 versions of those fonts (PF2 is a proprietary GRUB bitmap font format)
* `grub_themes/`: contains the pretty SQ GRUB theme.
* `syslinux-splash.png`: contains the pretty SQ SYSLINUX/ISOLINUX background.

Live system related stuff:

* `live_scripts/`: All the files in this directory (except README.md) will be copied into `/usr/local/bin/` on the live system image, putting them in-$PATH. Useful for utility scripts that should be available on the live system.
* `live_scripts/install_nfast_tools`: This script is run at startup (via /usr/local/bin/startup and /etc/rc.d/init.d/livesys), and builds the nfast kernel module in /opt/nfast/driver and runs the /opt/nfast/sbin/install script. This has to be done from the live system because the install script behaves differently when no PCI card is attached to the system. The contents of this file block entering an interactive shell, thanks to `nfast_block_shell` and a bash profile addition.
* `live_scripts/startup`: This script is run at startup (via /etc/rc.d/init.d/livesys). The contents of this file DO NOT block entering an interactive shell (they run async).
* `live_scripts/nfast_block_shell`: Blocks until the existence of the path /.nfast-configured, which is intended to be touched after NFast Tools have been installed with `install_nfast_tools`.

## Bootloader
BIOS boot uses SYSLINUX/ISOLINUX. SYSLINUX/ISOLINUX config lives at `/usr/share/lorax/live/config_files/x86/isolinux.cfg`.

EFI boot uses GRUB2. GRUB2 config template lives at `/usr/share/lorax/live/config_files/x86/grub2-efi.cfg`.
