---
title: Configuring SMB on TrueNAS
description: How to enable the SMB service and create a share
published: true
date: 2024-11-13T22:43:05.967Z
tags: 
editor: markdown
dateCreated: 2024-05-14T17:24:19.265Z
---

# Configuring SMB on TrueNAS
## What is SMB?
SMB is the Server Message Block protocol, the default file-sharing protocol for Microsoft Windows, and also for macOS for the past several years.  It can also be used with Unix-y clients (Linux, FreeBSD, etc.), though NFS may be a better choice for those clients.  It's very likely you'll want to enable this service and at least one share on your system.

## TrueNAS SCALE
### Enabling the SMB Service
To enable the SMB service in TrueNAS SCALE, in the web GUI, go to System Settings (1) -> Services (2).
![scalesmb1.png](/configuration/scalesmb1.png)
Then check the box under **Start Automatically** for the SMB service, and move the switch under **Running** to start it.
![scalesmb2.png](/configuration/scalesmb2.png)
### Creating a shared folder
In order to create a shared folder, you must first have at least one user on your system that isn't an administrator.  If you don't already have such a user, you can create one under Credentials -> Local Users.  Once you have such a user, create a shared folder, in the web GUI, click on Shares (1), then the **Add** button next to **Windows (SMB) Shares** (2).
![scalesmb3.png](/configuration/scalesmb3.png)
Under **Path**, expand `/mnt`, then choose your pool (1), then click on Create Dataset (2)
![scalesmb4.png](/configuration/scalesmb4.png)
In the pop-up window, enter a name for the share, then click **Create**.  The dataset will be created, and the share name (in the **Name** field) will default to the name you gave the dataset.  Enter a **Description** if desired (it's optional), then click **Save.**  Your shared folder will be created.

## TrueNAS CORE
### Enabling the SMB Service
To enable the SMB service in TrueNAS CORE, in the web GUI, click on Services, then scroll as needed to find SMB in the list.  Check the box to start automatically, and flip the switch to turn on the service.
![coresmb1.png](/configuration/coresmb1.png)
### Creating a shared folder
In order to create a shared folder, you'll need to have already created a dataset to share.  Then, in the web GUI, click on Sharing (1), then Windows Shares (SMB) (2), then Add (3).
![coresmb2.png](/configuration/coresmb2.png)
Then navigate to the dataset you want to share, as shown below.  The share name will default to the name of the dataset, but you can change it if desired.  Enter a description if desired.  Then click **SUBMIT** to create the share.
![coresmb3.png](/configuration/coresmb3.png)