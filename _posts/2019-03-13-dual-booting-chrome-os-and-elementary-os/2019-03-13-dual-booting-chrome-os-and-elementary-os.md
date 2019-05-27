---
layout: post
title: "Dual Booting Chrome OS and elementary OS"
---

I recently decided to purchase a [Chromebook](https://en.wikipedia.org/wiki/Chromebook). Considering that it's pretty easy to find them for under $200, they seemed like a good deal, especially taking into account the new features that Chrome OS has been adding (namely, [the ability to install Android apps](https://support.google.com/chromebook/answer/7021273?hl=en) and [run Linux in an container via Crostini](https://chromium.googlesource.com/chromiumos/docs/+/master/containers_and_vms.md)). What is more interesting, however, is that Chromebooks allow for disabling verified boot: while Chrome OS is itself Linux-based, with this validation disabled we can also install a full-blown, "standard" Linux distribution on the device. With a bit of effort, I've managed to get this to work pretty well; to save others (and myself, if I end up having to redo this later!) time I felt like it might be useful to document my installation process here.

## Overview

To aid those from the internet who might stumble across this post, I'm going to lay out the goal of this process explicitly: **this post goes through the process of installing [elementary OS](https://elementary.io) 5.0 "Juno" _alongside_ Chrome OS on the *internal* storage of my Chromebook, a [Samsung Chromebook 3 "Celes" XE501C13-K02US](https://www.amazon.com/Samsung-Chromebook-XE501C13-K02US-Dual-Core-Charcoal/dp/B07G576WS1) (with a dual core "Braswell" [Intel Dual-Core Celeron N3060](https://ark.intel.com/content/www/us/en/ark/products/91832/intel-celeron-processor-n3060-2m-cache-up-to-2-48-ghz.html) processor, 4 GB of DDR3 RAM, and 32 GB of eMMC storage).** I can't be certain that this will work on your device, especially if your configuration deviates from mine, but hopefully it can still be useful in some part. I'll try to document places where I had to "experiment" or piece together things to work the way I wanted it to. If you follow this guide exactly, you should end up with a Chromebook that boots Chrome OS with a 5 GB "user state" partition (at the time of writing sufficient to fit Chrome OS, a very lightly used Android runtime and Linux container with a gigabyte or two left over) and an elementary OS install with a ~20 GB `/` partition and a 500 MB `/boot` partition (which ends up about a third full after the install). With some luck, and possibly some small changes, you might be to make this work with other Intel-based Chromebooks or Linux distributions. Finally, you're going to have a flash drive that's at least 2 GB large for elementary OS. If you mess up, you're going to need an 8 GB flash drive on hand for the [recovery image](https://support.google.com/chromebook/answer/1080595?hl=en).

{% include aside.html content="A word about my device in particular: the reason I have provided an Amazon link above is that it seems like my model is a recent, minor design refresh to the [Samsung Chromebook 3 XE500C13-K02US](https://www.samsung.com/us/computing/chromebooks/under-12/chromebook-3-11-6-xe500c13-k02us/). It seems that these models are almost identical (though, I much prefer the external design of the new one), to the point that they share the \"Celes\" codename, and it seems that this revision is so new and so minor that Samsung hasn't bothered to put it on its website. I don't think anyone has done a deep dive on this yet, but I'm reasonably confident that their internals are extremely similar."%}

### Partition Layout

Chromebooks require a very specific partition layout; any changes to this (and I mean *any*, as we'll see) will cause Chrome OS to fail to boot with a "Chrome OS is missing or damaged" error. Since our goal is to have both elementary OS and Chrome OS side-by-side, this means that we are limited in the changes we can make–for example, we cannot add partitions, nor can we remove them. Thankfully, the disk format is [well documented](https://www.chromium.org/chromium-os/chromiumos-design-docs/disk-format):

![The disk layout of Chrome OS, showing the twelve partitions.](ChromeOSDiskLayout.png)

In logical order, the partitions are:

<div class="table-wrapper">

|-----------|------------|
| Partition | Label      |
|:---------:|------------|
| 1         | STATE      |
| 2         | KERN-A     |
| 3         | ROOT-A     |
| 4         | KERN-B     |
| 5         | ROOT-B     |
| 6         | KERN-C     |
| 7         | ROOT-C     |
| 8         | OEM        |
| 9         | reserved   |
| 10        | reserved   |
| 11        | RWFW	     |
| 12        | EFI-SYSTEM |
|------------------------|

</div>

`KERN-A` is the kernel for Chrome OS, while `ROOT-A` is the root filesystem; `KERN-B` and `ROOT-B` fulfill a similar function and are used by the autoupdate system. These, along with most of the other partitions, don't need to be touched; we will leave them as they are (except for `RWFW`, which the firmware script will modify). The three partitions we care about are `STATE`, `KERN-C`, and `ROOT-C`; the first stores user data and is typically the largest partition, while the other two are currently unused in Chrome OS but are theoretically supposed to serve a purpose similar to that of the other `KERN` and `ROOT` partitions.

With the current partition map, there is little space for our elementary OS installation. To make this work, we will shrink `STATE` (which unfortunately also wipes it) and expand `KERN-C` and `ROOT-C` into the newly freed space, with `KERN-C` being mapped to `/boot` in our elementary OS installation and `ROOT-C` becoming `/`. With this out of the way, let's get on with actually performing the installation.

## Installation Steps

The installation consists of five main steps: basic setup, updating the firmware of the device, partitioning it, installing elementary OS, and fixing GRUB so we can boot the installation. **The first and third steps _will_ wipe your device. If you something goes wrong in these steps, you will likely lose data as well.** Be careful, and keep backups! There's a section at the end for recovering from mistakes (though, these generally involve data loss).

### Setup

The first thing you will have to do is enable [Developer mode](https://www.chromium.org/chromium-os/developer-information-for-chrome-os-devices/generic) on your Chromebook. On mine, the keyboard combination for this is <kbd>Power</kbd>+<kbd>Esc</kbd>+<kbd>Refresh</kbd> (the third function key) to get to recovery mode, then <kbd>Ctrl</kbd>+<kbd>D</kbd> ([here's](https://chromium.googlesource.com/chromiumos/docs/+/master/debug_buttons.md#Firmware-Keyboard-Interface) a list of these shortcuts). **This will wipe your device, as well as make it less secure.**

Once you've done that, go ahead and boot the computer and connect it to Wi-Fi. It's probably not worth going further than this in the setup process, since we will be wiping the computer again, but you are free to do so if you'd like.

### Update your firmware

Depending on your Chromebook model, you may need to install [an updated firmware](https://mrchromebox.tech/#fwscript) to be able to boot Linux ([the GalliumOS wiki](https://wiki.galliumos.org/Hardware_Compatibility) is good place to check to see whether you need to do this); this is a shell script that will do the steps necessary for this process. Assuming you have enabled Developer mode and have just passed the Wi-Fi setup screen, you can access a shell by pressing <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Forward</kbd> (the second function key): as the message will tell you, the username is `chronos`. If you are already logged in to Chrome OS just press <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>T</kbd> and type `shell` like you normally would. Then download and run the script:

```console
$ curl -LO https://mrchromebox.tech/firmware-util.sh
$ sudo bash firmware-util.sh
```

At the prompt, type `1` and press enter to "Install/Update RW_LEGACY Firmware"; it's your choice how you want the boot order to be. Pick one and let it finish, then type `q` to exit. You don't have to reboot; just go to the next step.

### Partition the disk

Before we partition the disk, you will need to decide how much space you want to allocate to each partition. I gave `STATE` 5 GB, `KERN-C` 500 MB, and `ROOT-C` the remainder (about 20 GB), and the following commands will use this layout; however, you may choose different sizes based on your needs. **These steps will wipe your Chromebook's data**.

{% include aside.html content="There are [a number](https://github.com/ethanmad/chromeos-resize) [of tools](https://chrx.org) that supposedly make this process simpler, but I could not get any of them to work because they seemed to make faulty assumptions about my partition layout. If they work for you, you can try using them, but otherwise you'll have to do some math by hand."%}

The command we will be using throughout this section is `cgpt`, which handles partitions on Chrome OS. **It is quite important that Chrome OS is the _only_ thing that touches your partitions**–failure to adhere to this so has caused a "Chrome OS is missing or damaged" error for me every time. `cpgt` requires superuser privileges to run, and always takes as an argument the device it will work on. For eMMC backed-computers, this will likely be `/dev/mmcblk0`; if you have an SSD this might be `/dev/sda`. First, list the partitions on the disk:

```console
$ sudo cgpt show /dev/mmcblk0
   start        size    part  contents
       0           1          PMBR (Boot GUID: BB000A0D-0001-0EB4-CD10-AC3C0075F4C3)
       1           1          Pri GPT header
       2          32          Pri GPT table
 8704000    52367312       1  Label: "STATE"
                              Type: Linux data
                              UUID: 56FACA9B-B561-5E4E-B225-67FF9101E64A
   20480       32768       2  Label: "KERN-A"
                              Type: ChromeOS kernel
                              UUID: 0981A5B2-C79A-E145-802A-DEA62009F8DA
                              Attr: priority=1 tries=0 successful=1 
 4509696     4194304       3  Label: "ROOT-A"
                              Type: ChromeOS rootfs
                              UUID: BDBF49E7-B855-D04E-B09E-FDF2FE9E72A9
   53248       32768       4  Label: "KERN-B"
                              Type: ChromeOS kernel
                              UUID: F7D51A74-2D18-5649-B352-B3C779269C71
                              Attr: priority=0 tries=15 successful=0 
  315392     4194304       5  Label: "ROOT-B"
                              Type: ChromeOS rootfs
                              UUID: D47975C2-3B96-F54A-AAA7-394802C5BED2
   16648           1       6  Label: "KERN-C"
                              Type: ChromeOS kernel
                              UUID: 64A4302A-7FFD-0543-9ADD-004D20919615
                              Attr: priority=0 tries=15 successful=0 
   16649           1       7  Label: "ROOT-C"
                              Type: ChromeOS rootfs
                              UUID: 09C55DBA-67B7-A14A-BC05-C93124403597
   86016       32768       8  Label: "OEM"
                              Type: Linux data
                              UUID: DB2D7C8A-7F32-1F46-BC47-313EAB282E35
   16450           1       9  Label: "reserved"
                              Type: ChromeOS reserved
                              UUID: 531EB8EE-4240-2D44-A85F-27521C3934B0
   16451           1      10  Label: "reserved"
                              Type: ChromeOS reserved
                              UUID: F2E20DFD-8967-DE4F-96A2-F780936B29A2
      64       16384      11  Label: "RWFW"
                              Type: ChromeOS firmware
                              UUID: 9AC55AEB-1593-C247-BC54-9C9D9998990B
  249856       65536      12  Label: "EFI-SYSTEM"
                              Type: EFI System Partition
                              UUID: CDCA82E4-547E-BE4E-AD55-C3B5EFD5DF79
                              Attr: legacy_boot=1 
61071327          32          Sec GPT table
61071359           1          Sec GPT header
```

The partitions are listed with their labels, partition number, start block, and size. Blocks are 512 bytes in length; notice that while the table is listed in *logical* order, it is not in *physical* order–the blocks are actually in a different order on disk (this, by the way, seems to be what trips up the scripts above). Let's put the partitions in physical order:

<div class="table-wrapper">

|------------------------------------------|
| Number | Label      | Start   | Size     |
|:------:|------------|--------:|:--------:|
| 11     | RWFW       | 64      | 16384    |
| 6      | KERN-C     | 16648   | 1        |
| 7      | ROOT-C     | 16649   | 1        |
| 9      | reserved   | 16450   | 1        |
| 10     | reserved   | 16451   | 1        |
| 2      | KERN-A     | 20480   | 32768    |
| 4      | KERN-B     | 53248   | 32768    |
| 8      | OEM        | 86016   | 32768    |
| 12     | EFI-SYSTEM | 249856  | 65536    |
| 5      | ROOT-B     | 315392  | 4194304  |
| 3      | ROOT-A     | 4509696 | 4194304  |
| 1      | STATE      | 8704000 | 52367312 |
|------------------------------------------|

</div>

Luckily, the `STATE` partition (which is the first one, logically) is right at the very end physically, which means we can shrink it easily. To do this, we can use `cgpt add`, which takes the partition number with `-i`, starting block with `-b`, and size in blocks with `-s`. For this, we will keep its starting point, block 8704000, the same and shrink the partition down to 10485760 blocks (5 GB):

```console
$ sudo cpgt -i 1 -b 8704000 -s 10485760 /dev/mmcblk0
```

Unfortunately, `KERN-C` and `ROOT-C` are wedged in the middle of other partitions, making it impossible to increase their size without overwriting the subsequent ones. However, we can just change their base to fall right after `STATE` and expand them in the new space we have created. Thus, `KERN-C` (the sixth partition) will now start at block 8704000+10485760+1=19189761 and span 1048576 blocks, while `ROOT_C` (the seventh) starts at 19189761+1048576+1=20238337. `ROOT-C`'s size is the amount we shaved off of `STATE` minus the size of `KERN-C`, or (52367312-10485760)-1048576=40832976. In `cgpt`, this translates to

```console
$ sudo cgpt -i 6 -b 19189761 -s 1048576 /dev/mmcblk0
$ sudo cgpt -i 7 -b 20238337 -s 40832976 /dev/mmcblk0
```

With this done, you can reboot the system:

```console
$ sudo /sbin/reboot
```

When Chrome OS boots up, it will repair itself, taking about five minutes (there's a small, somewhat accurate timer in the top left that tracks progress). Once it's done booting, drop back into a shell; we now need to format the new partitions as ext4 so they are usable from Linux.

{% include aside.html type="Warning" content="Doing this from Linux (e.g. using GParted) seems to cause Chrome OS to throw the \"Chrome OS is missing or damaged\" error, hence the use of Chrome OS." %}

Once you're back in a shell, use `mkfs.ext4` to initialize the partitions we will be using for elementary OS (`KERN-C` and `ROOT-C`, which are the sixth and seventh partitions in `/dev/mmcblk0` for my Chromebook):

```console
$ sudo /sbin/mkfs.ext4 /dev/mmcblk0p6
$ sudo /sbin/mkfs.ext4 /dev/mmcblk0p7
```

### Installing elementary OS

You're now going to have grab your bootable installer USB. Plug it into the computer and reboot:

```console
$ sudo /sbin/reboot
```

On startup, press <kbd>Ctrl</kbd>+<kbd>L</kbd> this time; this will start legacy boot. Depending on the options you selected earlier when you installed the new firmware, the USB will either automatically boot or you will have to quickly press <kbd>Esc</kbd> and select your USB device from the boot menu.

{% include aside.html content="You may get an error saying \"graphics initialization failed\"; apparently this is a bug that can be fixed [by typing \"help\" and pressing enter twice](https://ubuntuforums.org/showthread.php?t=1594003)." %}

Once you have booted, start the installer.

{% include aside.html content="On my trackpad, clicks didn't work at all; you can either use an external mouse or tap to click to get around this. If you do accidentally click, you may end up in a state where it's no longer possible to click; in that case just switch to a different console with <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>F1</kbd> (Forward) and back with <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>F7</kbd> (Brightness up)." %}

{% include aside.html type="Update" content="The fix for this appears to have been merged into a later kernel, somewhere around 4.16. Updating the kernel (or [patching](Samsung-Chomebook-Patches.zip), building, and installing the kernel yourself) should make the trackpad work correctly." %}

Go through the setup as usual, until you reach the bit where it asks you about your installation type. Make sure to select the option for "Something else" here:

![Ubiquity installer, showing the "Installation type" screen. The "Something else" option is selected.](InstallationType.png)

This will take you to the partition editor. Here, we need to modify the `KERN-C` and `ROOT-C` partitions. Click on them and them change them to have a "Ext4 journaling file system" (do not format the partition–leave the checkbox unchecked); in addition, change the mount point for the first one to `/boot` and the second to `/` (this might cause the installer to prompt about performing a resize, or something like that; out of fear I just hit back on the dialog and it seemed to work out). Finally, change the device for boot loader installation to where `KERN-C` is. In the end, it should look something like this:

![Ubiquity installer, showing the continuation of the "Installation Type" screen. /dev/mmcblk0p6 is to be used as /boot, and /dev/mmcblk0p7 is to be used as /; both are formatted as ext4. The device for boot loader installation is /dev/mmcblk6.](InstallationTypeContinued.png)

Going to the next step will probably cause some warnings to pop up about not formatting; just skip through those. Continue with the rest of the installation as normal, however, hold off on rebooting when the installation completes.

### Fixing GRUB

The installer exits in a state which is unfortunately still not bootable, so we will have to fix this ourselves. Launch a terminal, and create `/mnt/boot`:

```console
$ sudo mkdir /mnt/boot
```

Mount the boot partition, `KERN-C`, there (*not* the whole device, just the partition):

```console
$ sudo mount /dev/mmcblk0p6 /mnt/boot
```

We need to install GRUB to this partition now. We will need to pass it the `--boot-directory` flag to tell it where our boot directory is, and possibly `--force` if it complains about blocklists being unreliable. In all, the command should look something like this:

```console
$ sudo grub-install --boot-directory=/mnt/boot /dev/mmcblk0 --force
```

Notice that we did not use the partition here; this is for the *entire* device. This will set up the correct MBR records and allow us to boot straight into elementary OS next time we perform a legacy boot. Reboot your computer, remove the Live USB, and press <kbd>Ctrl</kbd>+<kbd>L</kbd> at startup: the Chromebook *should* go directly into Linux. Pressing <kbd>Ctrl</kbd>+<kbd>D</kbd> should take you to Chrome OS instead (do check this: Linux is less picky about booting, and these steps can easily lead to a broken Chrome OS install even when elementary OS boots fine). If this doesn't work, see below for troubleshooting.

## Troubleshooting

You've done something wrong and now Chrome OS doesn't work, or you're being dropped into a `GRUB>` prompt. Here's some stuff that might help.

### Chrome OS Recovery

If you are getting an error that Chrome OS is missing or damaged when trying to boot it, then you need to use [Chrome OS Recovery](https://support.google.com/chromebook/answer/1080595?hl=en). This will reset your Chromebook to its original out-of-box state (*wiping your data*), which also makes it useful as a way to go back to single-boot Chrome OS if you so desire.

Supposedly, there's a [Chrome Extension](https://chrome.google.com/webstore/detail/chromebook-recovery-utili/jndclpdbaamdhonoechobihbbiimdgai) that does the work of flashing this to a flash drive for you: don't bother with it; it didn't work for me and it's really not necessary, since you can just download and flash the image yourself. The list of recoveries that the tool uses is [here](https://dl.google.com/dl/edgedl/chromeos/recovery/recovery.conf); look for your device in this, grab the URL, and download it by hand. You can then unzip that firmware and directly image it (via `dd`, if you have it) to a flash drive. I'd keep this around for a bit, until you're sure that your installation works fine and you don't want to go back to a factory install.

### I messed up when installing Linux

Does your Chrome OS still boot? If so, just wipe your `KERN-C` and `ROOT-C` partitions with `mkfs.ext4` and try again, assuming that your partitions are the correct size; if all goes well this should preserve your Chrome OS installation. If not, go back another step–this will unfortunately erase your Chrome OS data. If that doesn't work, either, use the recovery to start from the beginning.

### I can't boot into Linux

If you're getting a black screen after pressing <kbd>Ctrl</kbd>+<kbd>L</kbd>, getting stuck at "Booting from hard disk", or a `GRUB>` prompt, it's likely that the GRUB fix described above didn't work. Double check to make sure you installed the boot loader to the correct partition, as well as did `grub-install` to the right place. Even if you have installed your Linux correctly, I have seen the system mess up sometimes after booting from a Live CD, or updating the Linux kernel–just run the GRUB install step again and this should take care of the issue.
