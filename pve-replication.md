---
title: Replicate Proxmox ZFS Pool to TrueNAS
description: Replicate contents of the Proxmox root ZFS pool to a TrueNAS SCALE system
published: true
date: 2024-01-11T12:02:43.061Z
tags: 
editor: markdown
dateCreated: 2024-01-05T14:35:57.828Z
---

# Proxmox -> TrueNAS Replication
## Introduction
Proxmox includes tools (i.e., [pve-zsync](https://pve.proxmox.com/wiki/PVE-zsync), [Storage Replication](https://pve.proxmox.com/wiki/Storage_Replication)) to send snapshots of your virtual machines to another system using ZFS replication.  Neither of these, however, back up the Proxmox system itself.  This guide will configure your Proxmox host and TrueNAS SCALE server to periodically back up the root ZFS pool on your Proxmox host to your NAS.

These instructions will result in a backup of everything on your Proxmox root pool, including any virtual machines you have running on the default `local-zfs` storage.  You can customize the snapshot task if this isn't the desired behavior.

## Requirements
You must have installed Proxmox Virtual Environment using ZFS.  The pool layout (stripe, mirror, RAIDZn) doesn't matter, but it must be a ZFS pool.  Your Proxmox must must also be set up to allow root login via SSH using public-key authentication.  This guide was developed using TrueNAS SCALE 23.10.1.  While the same basic procedure will work with TrueNAS CORE, or other versions of TrueNAS SCALE, some of the options may be in different places.

## Create SSH connection
TrueNAS performs ZFS replication over a SSH connection.  To set this up, in the TrueNAS web UI, go to **Credentials -> Backup Credentials**.  Under **SSH Connections**, click **Add**.  In the new window that pops up, enter:

* Name: Whatever you want to call this connection--I use the hostname of the host I'll be connecting to.
* Setup Method: Manual
* Host: The hostname or IP address of your Proxmox server
* Port: The SSH port on your Proxmox server; default is 22
* Username: root
* Private Key: Generate New
* Under the Remote Host Key field, click **Discover Remote Host Key**

If your Proxmox hosts are in a cluster, the `authorized_keys` file will be shared among them, so you'll only need to generate one private key.  You'll still need to set up a SSH connection for each host, but rather than generating a new private key, just choose the first one from the drop-down.

Then click **Save**.
## Prepare your Proxmox host
### SSH Key
If you're configuring the second (or subsequent) host in a Proxmox cluster, the SSH key will already be present, so you can skip this step.  If not, follow these steps:

Still in the TrueNAS web UI, under **SSH Keypairs,** click on the one you just created.  In the window that comes up, copy the **Public Key.**

Then log in to your Proxmox host via SSH or its web console.  Paste the public key into `/root/.ssh/authorized_keys`.
### ZFS Snapshots
ZFS replication operates on snapshots.  When TrueNAS is replicating to or from another TrueNAS system, it's able to directly manage snapshots on the remote system.  With a non-TrueNAS system, however, it isn't, so you'll need to create and manage them on the Proxmox host itself.  This guide uses `zfSnap` to do this.

Log into the shell on your Proxmox host and install zfSnap by running `apt install zfsnap`.  Then create your first snapshot by running `zfSnap -a 5d -r rpool`.  You can confirm the snapshots are present by running `zfs list -t snapshot`.

Next, set up a cron job to create these snapshots on a schedule, and to delete old snapshots once they're older than the lifespan you've given them.  Run `EDITOR=nano crontab -e` to edit your crontab (if you like `vi`, you can omit the `EDITOR=nano` portion of this command).  Enter:
```
58 * * * * /usr/sbin/zfSnap -d -a 5d -r rpool
```
Then save and exit the editor.  This will run every hour at 58 minutes after the hour (so that they'll be present for the replication task that runs on the hour), creating a recursive snapshot of `rpool` with a lifespan of five days, and delete any snapshots that are older than that lifespan.

If you'd like to change any of these details, consult the zfSnap help (`zfSnap --help`) fore more information on its options.
## Set up the replication task
### Create a destination
Replication tasks on TrueNAS need an empty destination, which needs to be its own dataset.  You can create one under **Datasets** in the TrueNAS web UI.
### Create the replication task
Still in the TrueNAS web UI, go to **Data Protection**, and under Replication Tasks, click **Add.**  In the window that comes up, enter:

* Source Location: On a Different System
* SSH Connection: The connection you set up previously
* Skip Source for a moment, and under "Include snapshots with the name," set it to "Snapshot Name Regular Expression."  For the regular expression, enter `.*`.  This will replicate all snapshots on the Proxmox host.
* Now return to Source, expand the folder, and check the box next to `rpool`.
* Check the box for Recursive.
* To the right, under Destination, navigate to the dataset you created above.
* Scroll down and click **Next.**

At the next screen, set your desired schedule.  I'll be running this replication hourly.
### Test
Now it's time to make sure this works.  You should be back at the Data Protection screen in the TrueNAS web UI, and should have a replication task with a State of "Pending."  Click the Play button next to that task, and in the window that pops up, click **Continue.**  If it doesn't return an error in a moment or two, you should be good to go.
