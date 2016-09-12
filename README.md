_Please read this entire file before asking for help or reporting bugs_

# Foreword

Flashboot is a system built by Damien Miller and others as an adaptation 
of OpenBSD that's more suited for small flash-based hardware. 
For most applications you don't have to compile it on you own, 
you just put the binary release on a flashcard and you're set (somewhat simplified).

Flashboot is not OpenBSD and it's not a official port. We are using the word
OpenBSD because we don't change any source code other then the scripts that
are buildning the ram disk kernel and some basic defaults configurations. The
kernel and the userland is untouched but stripped to the minimum to make
Flashboots fotprint as smal as possible.

Don't turn to OpenBSD mailing list for support since this project is
unofficial. Please submit bug reports via the
[GitHub Issue Tracker](https://github.com/openbsd/flashboot/issues).

# Introduction

This is a small infrastructure to build minimal OpenBSD installations suitable
for booting of flash and USB devices. This is derived from the scripts and
tools used to build the OpenBSD installation media.

The system builds a ram disk kernel (bsd.gz) with a reasonably complete set of
daemons and utilities. This file should be installed into the root directory
of the boot device. Note that this infrastructure only builds the kernel
image, it is beyond its scope to install it onto the flash device, or to
prepare the flash device for booting the kernel. If you are playing with this
stuff, you should know how to do this anyway.

When the flashboot kernel boots, it will attempt to mount the flash device as
/flash and extract any tarball files matching /flash/*.tgz into the root
directory. This mechanism is intended to include the system configuration that
the host will use, though it may be used to deploy updated or new binaries
too. The configuration parameters accepted include most of the standard
OpenBSD /etc/ files, including /etc/hostname.* and a useful subset of
/etc/rc.conf. See the initial-conf/rc script for details. As well as
extracting /flash/*.tgz, the init script will copy any files under the
/flash/conf/ directory to the root too - I find this a little more convenient
for small config files that frequently change (e.g. pf.conf).

In the absence of a tarball or directory-based config, the system is
configured with a basic initial configuration. The first Ethernet interface is
brought up as 192.168.0.1 and the console is available (login: "root"
password: "password"). The intention is to make it easy to load a real config
from a freshly imaged flash card.

The assumed platform is a Soekris NET4501 (www.soekris.com), but is very
easily changed by selecting a different kernel config and adjusting or
renaming the initial-conf/hostname.sis0 file.

# Installation

To install flashboot onto your media, you will need to format it and make it
bootable first. Read the fine boot(8) and installboot(8) man pages for
instructions on how to prepare the media.

Once the media is setup as bootable, you can just copy the kernel image to
/bsd on the mounted device. You should also create a /conf directory to store
your custom configuration - files under here are copied into the ramdisk at
system boot time. After startup the boot device is mounted as /flash in the
root of the ramdisk, so you can make changes at runtime there.

To influence the boot loader, you can create a /etc/boot.conf file on the root
of the bootable media (*not* /conf/etc/boot.conf). You can use the standard
boot.conf(5) options, such as changing console device and speed here.

# Building

(if the following sounds too hard, please use the binary distribution of
flashboot, available from the website)

A built release of OpenBSD is required for this infrastructure to work. To
save space, you will probably want to turn off compile-time options such as
Kerberos support. The current version of flashboot requires that you have
built the entire system as dynamically linked, i.e. set STATIC= in mk.conf.
Previous flashboot versions used crunchgen, but fully-dynamic saves 1.5Mb.

Here are the basic steps you need to take.

1. Read and understand "man release" 
2. Build a release (using build-release.sh)
3. Build a kernel (using build-kernel.sh)
4. Install obj/bsd.gz to the root of the boot device as bsd
5. Ensure that installboot has been run on the boot media

First, make sure all building scripts has the execute bit set for you and all
other (chmod ugo+x build-*).

Next, edit release.sh and change SHORTREL and LONGREL to match the version you
want to build. You should also change URLBASE to match a mirror near you.

build-release will download, patch and build userland automatically

This build can take a very long time. Make sure you run this in the 
flashboot-directory and with doas since the script needs to be able
to mount devices.

    # doas ./build-release.sh

When the building is done you should have a fully dynamic release in
sandbox. Now it's time to make your kernel. The
build-kernel.sh takes a kernel-config as a parameter. Currently
COMMELL-LE564, GENERIC-RD, NTFS, SOEKRIS4501, SOEKRIS4521,
SOEKRIS4801, SOEKRIS5501 and WRAP12 configs are provided. You can
easily make your own by copying one of the existing. 
    
    # doas ./build-kernel.sh WRAP12

The kernel is then stored under the obj/ directory. The use of
a gzipped kernel is preferred. The bootloader automatically extracts
it during boot. It saves space on the flash and transfer time during
upgrade.


# Customisation

The system supports customisation in three different ways.

1. Any *.tgz file that you add to the root of the flash-card (/flash) or
the release-dependent subdirectory (e.g.  /flash/5.3 for a 5.3 kernel) is
automatically extracted to the ramdisk during boot. This is useful for small
extensions or configuration that you can distribute in a single tgz-package.
Files can also be added to the /flash/conf directory, they are then
automatically copied (not extracted) to the respective location in the
ramdisk during boot, e.g. /flash/conf/etc/myname will end up as /etc/myname
when the system has booted.

2. There is also an option in rc.conf to create a second ramdisk in
/usr/local. By default this is not done at all to save memory on the device.
If activated this will also extract any *.tgz files located in /flash/pkg and
the release-dependent subdirectory (e.g. /flash/pkg/5.3 for a 5.3 kernel)
to /usr/local. This provides an excellent way to add packages directly from the
OpenBSD ftp-server without needing to expand the original ramdisk. The
rc-script will even remove some unnecessary files from the packages such as
man-pages and files needed only for compiling.

3. Recompile a kernel that suits your needs and that includes or excludes the
binaries and libs that you need for your application. Change the list to
include extra files. If you have decided to go down this road you might just
as well modify the default configuration located under the initial-conf
directory.

The easiest way to add configuration is to add files and folders to
initial-conf and add list.custom in Flashboot root folder where instructions
to the build script for each file is provided. Look at the included list file
to get correct syntax.

The first two alternatives can be done without having to recompile the
distribution and should be sufficient for most needs.

Kernel sizes are a problem when piggybacking a ramdisk blob. Kernel+ram disks
larger than 16MB needs to increase the NKPTP in the kernel config. This is
already done for all the kernels and you shouldn't have to do anything as long
as you stay under 32MB. The second problem is that kernels larger than about
14MB will use up all the ISA DMA memory, for this reason ISA DMA is disabled.
This leads to the final problem: if isadma is disabled, then things that
attempt to use it (e.g. floppy disk access) will panic the kernel. In the end
the best solution is to try to keep your kernels small.

This infrastructure does a couple of things to save space, but is not at all
ferocious as I stopped tweaking once I reached my target (bsd.gz < 5Mb). To
save space, only a couple of term{cap,info} entries are transferred (see
TERMTYPES in the Makefile).


# Creating Bootable CD

To create an iso image suitable for booting and running flashboot on a x86 PC,
first read the build instructions above. Step 1 and 2 must be successful and
in step 3 use the build-livecd.sh script with the GENERIC-RD kernel config.

    # doas ./build-livecd.sh GENERIC-RD

The build-live_cd.sh script will create the directory live_cd before creating
the image. Everything in this directory will be included on the cd. The files
cdbr, cdboot etc/boot.conf is required for booting. To customize the cd with
additional data, create the directory live_cd under the sandbox directory in
advance and populate with data and modify rc.{init|more} as needed.

The build-livecd.sh will use the default flashboot config files and make
modifications prior to ramdisk creation. Any customisation to ramdisk setup
should be done by editing build-livecd.sh.

If everything worked then you should find a new iso image in
obj/live_cd{version}.iso. Without any customisation the iso is roughly 21MB in
size. Finally... burn the iso with your favourite cd burning program.


# Support and bug reporting

Please submit bug reports via the
[GitHub Issue Tracker](https://github.com/openbsd/flashboot/issues).
