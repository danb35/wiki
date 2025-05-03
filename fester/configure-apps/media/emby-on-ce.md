---
title: Emby Server on TrueNAS CE
description: Installation and basic configuration of Emby Server on TrueNAS Community Edition
published: true
date: 2025-05-03T18:51:22.647Z
tags: 
editor: markdown
dateCreated: 2025-05-03T18:51:22.647Z
---

# Emby Server
## Introduction
Emby is a popular piece of software that allows you to organize and consume your media using a variety of devices.  Like Plex, it is free-to-use but closed-source software, and certain features require a paid subscription.
## Prerequisites
* You are running TrueNAS SCALE Electric Eel (24.10) or later.  Due to drastic changes in the apps ecosystem, this guide is not expected to work with earlier versions of TrueNAS.
* You have already initialized the Apps system by choosing a pool.
* You've created at least one non-admin user on your system.
# Preparation
## Create a media dataset(s)
In the TrueNAS web UI, click on Datasets.  Select your pool and click **Add Dataset.**  Call it `media`, and set the Dataset Preset to **Apps.**  Click **Save.**
![add-dataset-media.png](/apps/media/add-dataset-media.png)
Select the newly-created `media` dataset and click **Add Dataset.**  Name this one for one type of media you'll have in your library (e.g., `movies`), set the Dataset Preset to **Apps,** and click **Save.**  In the pop-up window asking you to set an ACL for this dataset, click **Return to pool list.**  Repeat for any additional media types (e.g., TV, music, etc.).
## Create an Emby dataset
Still on the datasets page, select your pool again.^[It's recommended that your apps live on an SSD pool, and generally recommended that app data of this sort do so as well.  If you've set up a separate SSD pool for your apps, select that pool here.]  Click **Add Dataset** and call the new dataset `emby-data`.  Again, set the Dataset Preset to **Apps** and click **Save.**
![add-dataset-embydata.png](/apps/media/add-dataset-embydata.png)
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
## Install Emby
In the TrueNAS Web UI, browse to Apps -> Discover Apps and find Emby Server.  Click **Install**.

Under Network Configuration, check the box for Host Network.

Under Storage Configuration, set **Emby Config Storage** to a type of Host Path, and select the `emby-data` dataset.
![emby-config-embydata.png](/apps/media/emby/emby-config-embydata.png)
Then, next to **Additional Storage,** click Add.  Set the Type to Host Path, set Mount Path to `/mnt/media`, and browse to your media dataset under Host Path.
![emby-config-data.png](/apps/media/emby/emby-config-data.png)
Then scroll to the bottom and click **Install.**  Wait a few moments for the application to deploy.

## Configuration
Once Emby deploys, click the Web UI button to open it.^[If you've enabled the HTTP->HTTPS redirect for your TrueNAS web GUI, this won't work.  Instead, browse to `http://TRUENAS_IP:8096`.]  You'll be greeted by the Emby welcome page.

Chose your language, and enter a username and password for the admin account.  At the next screen, click **+ New Library.**
![emby-new-library.png](/apps/media/emby/emby-new-library.png)
On the next screen, choose a content type (I'll use Movies for this example).
![emby-new-library2.png](/apps/media/emby/emby-new-library2.png)
Then click the + next to Folders.
![emby-select-path.png](/apps/media/emby/emby-select-path.png)
Browse to `/mnt/media/movies`, then click **Ok.**  Click **Ok** again to add the library.  Repeat as needed for any other media types.  When you're finished adding libraries, click **Next.**  Keep clicking **Next** to proceed through the rest of the setup, and then sign in as the admin user you created.
## Test
Jellyfin will take some time to scan the media in your media dataset, and you'll see items show up on your dashboard as it finds them.  Click on one of them and make sure it plays.
![emby-dashboard.png](/apps/media/emby/emby-dashboard.png)
# Conclusion
These steps have left you with a basic Emby installation.  For further information and documentation, visit https://emby.media/.