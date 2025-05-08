---
title: Plex Media Server on TrueNAS CE
description: Installation and basic configuration of Plex Media Server on TrueNAS Community Edition
published: true
date: 2025-05-08T13:10:32.688Z
tags: apps, media
editor: markdown
dateCreated: 2025-05-03T15:32:11.299Z
---

# Plex Media Server
## Introduction
Plex Media Server is a popular piece of software that allows you to organize and consume your media using a variety of devices, from anywhere with Internet access.  It is free but closed-source software, and some features (including remote streaming) require a subscription.
## Prerequisites
* You are running TrueNAS SCALE Electric Eel (24.10) or later.  Due to drastic changes in the apps ecosystem, this guide is not expected to work with earlier versions of TrueNAS.
* You have already initialized the Apps system by choosing a pool.
* You've created at least one non-admin user on your system.
# Preparation
## Create a media dataset(s)
In the TrueNAS web UI, click on Datasets.  Select your pool and click **Add Dataset.**  Call it `media`, and set the Dataset Preset to **Apps.**  Click **Save.**
![add-dataset-media.png](/apps/media/add-dataset-media.png)
Select the newly-created `media` dataset and click **Add Dataset.**  Name this one for one type of media you'll have in your library (e.g., `movies`), set the Dataset Preset to **Apps,** and click **Save.**  In the pop-up window asking you to set an ACL for this dataset, click **Return to pool list.**  Repeat for any additional media types (e.g., TV, music, etc.).
## Create a Plexdata dataset
Still on the datasets page, select your pool again.^[It's recommended that your apps live on an SSD pool, and generally recommended that app data of this sort do so as well.  If you've set up a separate SSD pool for your apps, select that pool here.]  Click **Add Dataset** and call the new dataset `plexdata`.  Again, set the Dataset Preset to **Apps** and click **Save.**
![add-dataset-plexdata.png](/apps/media/add-dataset-plexdata.png)
## Configure your user
Browse to Credentials -> Users, click on your user, and click **Edit.**  Make sure that user's Auxiliary Groups contains `apps` (add `apps` if it isn't present), and that the SMB User box is checked.  Click **Save.**  Repeat this process for any other users you want to have access to the media share you're about to create.
![edit-user-apps.png](/apps/media/edit-user-apps.png)
![edit-user-smb.png](/apps/media/edit-user-smb.png)
## Create a media share
Browse to Shares in the UI.  Next to **Windows (SMB) Shares,** click **Add.**  Under Path, browse to the media dataset you created above, then click **Save.**
![add-share-media.png](/apps/media/add-share-media.png)
## Test
On a desktop client computer, connect to your NAS via SMB.  Log in as the user you edited above.  Open the `media` share, and copy a few files into it.  If this all works, you can proceed.
![media-share.png](/apps/media/media-share.png)
# Installation
## Install Plex
In the TrueNAS Web UI, browse to Apps -> Discover Apps and find Plex.  Click **Install**.

Under Network Configuration, check the box for Host Network

Under Storage Configuration, set **Plex Data Storage** to a type of Host Path, and select the `media` dataset.  Set **Plex Configuration Storage** to a type of Host Path, and select the `plexdata` dataset.
![plex-config-data.png](/apps/media/plex/plex-config-data.png)
![plex-config-plexdata.png](/apps/media/plex/plex-config-plexdata.png)
Then scroll to the bottom and click **Install.**  Wait a few moments for the application to deploy.

## Configuration
Once Plex deploys, click the Web UI button to open it.^[If you've enabled the HTTP->HTTPS redirect for your TrueNAS web GUI, this won't work.  Instead, browse to `http://TRUENAS_IP:32400/web`.]  You'll be greeted by the Plex page, which will ask you to log in to your Plex account.
![plex1.png](/apps/media/plex/plex1.png)

While that can bring a variety of additional features, we're going to skip it for now (you can come back and log in later if you choose).  Click the "What's this?" link at the bottom, and on the next page, click "Skip and accept limited functionality."
![plex2.png](/apps/media/plex/plex2.png)
You'll next see a page with a basic introduction to Plex.  Click "Got It!"
![plex3.png](/apps/media/plex/plex3.png)
On the next page, once you close the advertisement for Plex Pass, you'll be able to set up your server.  Give it a name and click Next.  The next screen is where you'll set up your media libraries.  Click the "Add Library" button.
![plex-setup.png](/apps/media/plex/plex-setup.png)
Select your library type (I'll use Movies for this example):
![plex-add-library.png](/apps/media/plex/plex-add-library.png)
Next click on "Browse for Media Folder," and browse to `/data/movies`.  Then click "Add," then "Add Library."
![plex-add-folder.png](/apps/media/plex/plex-add-folder.png)
Repeat for any other media types, then click Next.
## Test
Plex will take some time to scan the media in your media dataset, and you'll see items show up on your dashboard as it finds them.  Click on one of them and make sure it plays.
![plex-dashboard.png](/apps/media/plex/plex-dashboard.png)
# Conclusion
These steps have left you with a basic Plex installation.  You'll probably want to create an account on https://plex.tv to use more advanced features, including streaming your media to your devices while away from home (which requires a paid subscription).  Enjoy!
