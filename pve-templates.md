---
title: VM Templates on Proxmox VE
description: Simplify Creating VMs by Using Templates
published: true
date: 2024-09-22T18:27:27.737Z
tags: 
editor: markdown
dateCreated: 2024-09-22T18:27:24.851Z
---

## Introduction
Other than virtualization itself, VM templates are one of the greatest reasons to operate in a virtualized environment.  They allow you to create a new VM in a matter of seconds, just by cloning a template, rather than running through the installation process from scratch every time you want, say, a Ubuntu 24.04 VM.  If you expect to create more than one or two VMs for a particular guest OS, it'd be worth your time to create a template for that OS.

I prepared the following instructions mainly for my own reference, so I could read the steps rather than watching a 20+-minute video again.  In hopes that it will be helpful to others, I've posted it here.  I'm mainly following Jay's instructions from LearnLinuxTV; I'll link that video and other resources below.  The primary difference between that video and these instructions is the use of `virt-customize` to install the Qemu Guest Agent, which was suggested by a commenter on that video.

These instructions have been tested and confirmed to work with Debian 12, and Ubuntu 22.04 and 24.04.  Rocky Linux 9 doesn't quite work with these instructions; specifically, the Qemu Guest Agent doesn't appear to run.

## Instructions
The process I describe uses cloud images provided for many Linux distributions.  These images are designed to work with cloud-init, which Proxmox supports, allowing you to configure that system as desired.

### In the Proxmox VE web interface
Start in the Proxmox VE web interface, and create a VM.  Give it any ID number you like (I'd suggest a fairly large number, so it will appear at the bottom of your VM list), and a name indicating it's a template (for example, `ubuntu-2204-template`).  On the next tab, select "Do not use any media."  On the next tab, check the box for "Qemu Agent." On the next tab, remove the virtual disk (we'll add another one later).  On the CPU and memory tabs, choose low values--1-2 cores and 1024 MB are fine.  You can increase those in your cloned VMs if needed.  Set the network interface as you normally would (`vmbr0` by default).  Confirm creation.

Now, navigate to the Hardware tab of the newly-created VM, click the "Add" button at the top, and choose "CloudInit Drive" from the menu.  Choose the storage location to create it, and click "Add."

Next, navigate to the Cloud-Init tab for that VM.  Here, you'll be able to provide some configuration for your template.  You'll ordinarily want to select a username to create, set that user's password, and optionally set one or more SSH public keys for that user.  You can set DNS search domain and servers, if you want them to be different from your Proxmox VE system's settings.  You can also specify the use of DHCP under "IP Config," which is recommended.  Once you've made these settings, click "Regenerate Image" at the top.

### At the shell
The next several steps are done using the shell as the root user.  SSH into your Proxmox VE host if possible, or you can use the "Console" button in the web interface if you prefer.

* First, download the cloud image file
  * For Ubuntu 22.04, `wget https://cloud-images.ubuntu.com/minimal/releases/jammy/release/ubuntu-22.04-minimal-cloudimg-amd64.img`
  * For Ubuntu 24.04, `wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img`
  * For Debian 12, `wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2`
  * For Rocky 9, `wget https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2`
* If the filename doesn't already end in `.qcow2` (as it does with the Debian image), rename the file to change the file extension.  You can optionally shorten it as well.  E.g., `mv ubuntu-22.04-minimal-cloudimg-amd64.img ubuntu-22.04.qcow2`
* Set the VM to use a serial console.  `qm set <VM ID> --serial0 socket --vga serial0`
* Resize the image file.  As with the CPU and memory settings, you can increase this size in your cloned VMs if needed.  `qemu-img resize <filename>.qcow2 16G`
* Now we're going to modify the image file by installing the Qemu Guest Agent, which will allow Proxmox VE to see VM IP addresses, cleanly shut them down, etc.  `virt-customize -a <filename>.qcow2 --install qemu-guest-agent`
  * If that command fails because `virt-customize` isn't found, install it using  `apt install libguestfs-tools`
* If there are any other packages you want to install in your template, install them in the same way.
* Now you need to empty the `/etc/machine-id` file in the disk image.   `virt-customize -a <filename>.qcow2 --truncate /etc/machine-id`
  * `virt-customize` created this file when you installed the `qemu-guest-agent` package, but it needs to be removed.  If this isn't done, all VMs you create by cloning this template will be issued the same IP address by your DHCP server, which will result in IP conflicts on your network.
* Next, import this disk image into the VM you created above.  `qm importdisk <VM ID> <filename>.qcow2 local-zfs`  Replace `local-zfs` with the name of the storage system you want to use to store the VM disk image.

### Back to the Proxmox VE web interface
Now, there are a few more steps to take back in the Proxmox VE web interface.  Navigate back to the Hardware tab for the VM you created, and you'll see a disk marked as `unused0`.  Select that and click the Edit button.  If it's backed by SSD, check the box for "Discard".  Also check the "Advanced" box at the bottom of the window, and check "SSD emulation."  Then click OK.

Now navigate to the Options tab for that VM, and edit the Boot Order.  You'll need to check the box for `scsi0` (it isn't checked because you didn't create a disk when you created the VM), and move it to the desired place in your boot order.

Finally, it's time to convert this VM into a template.  To do this, right-click on it in the VM list and select "Convert to template."  Confirm this, and in a moment your template will be created.

## Using your template
Now that you have the template created, using it is simple.  Just right-click on the template and choose "Clone" from the context menu.  Enter a VM ID (or accept the default) and a name.  You can create either a linked or a full clone; consult the [Proxmox docs](https://pve.proxmox.com/wiki/VM_Templates_and_Clones) for more information on these.  Then click the "Clone" button.  Once that completes in a moment, you'll have a new VM created.  If needed or desired, you can increase the RAM or core count, or resize the disk, before you start it.
## References
* https://www.youtube.com/watch?v=MJgIm03Jxdo - This video, by and large, lays out the process I document above.
* https://www.youtube.com/watch?v=E7rv08ttv8k - This video also documents use of a script to create multiple templates
  * and https://www.apalrd.net/posts/2023/pve_cloud/ for the script itself.  The script includes Ubuntu releases through 23.10, but not 24.04, though it could be easily edited to add that release if desired.
* https://www.youtube.com/watch?v=shiIi38cJe4 - Similar in concept, but does more at the CLI than in the Proxmox VE web interface
* https://www.youtube.com/watch?v=3I3C4RfluEg
