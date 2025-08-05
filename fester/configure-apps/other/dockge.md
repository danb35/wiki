---
title: Dockge
description: Installing and using Dockge under TrueNAS
published: true
date: 2025-07-27T13:06:30.214Z
tags: 
editor: markdown
dateCreated: 2025-07-27T13:06:27.081Z
---

# Installing and using Dockge in TrueNAS
## Background
Since the release of 24.10, TrueNAS has given users the ability to deploy arbitrary Docker Compose stacks as a "custom app via YAML."  However, as of version 25.04, this feature is not particularly user-friendly, even leaving aside its unintuitive location in the GUI.  Two noted problems are that it strips any comments out of the YAML entered, and that it doesn't support the use of `.env` files.

Enter [Dockge](https://github.com/louislam/dockge), a "fancy, easy-to-use and reactive self-hosted docker compose.yaml stack-oriented manager."  Dockge gives you a clean and straightforward GUI to enter, edit, and deploy Compose stacks, storing those stacks in plain text on your pool, where you can easily back them up wherever you like.  It's available as an app in TrueNAS.
## Preparation
### Choose a Pool
If you haven't already, you'll need to choose a pool.  In the TrueNAS UI, go to Apps, click on Configuration, and select Choose Pool.  Select a pool where your apps will live and run.
### Create datasets
On this pool, you'll need to create a `docker` dataset, and within that, `data`, `dockge`, and `stacks`.  In the TrueNAS UI, go to Datasets, select your pool, and click Add Dataset.  Give it a name of `docker`, and leave the preset at `Generic`.  Click Save.  Under that dataset, repeat the same process to create `data`, `dockge`, and `stacks`.
## Install Dockge
In the TrueNAS UI, go to Apps, then click Discover Apps.  Search for Dockge, click on it, then click Install.

Under Network Configuration, set the port number to 5001.

Under Storage Configuration, set the type for Dockge Stacks Storage to Host Path, and choose the `stacks` dataset you created above.  For Dockge Data Storage, again set the type to Host Path, and choose the `dockge` dataset you created above.

Scroll to the bottom and click Install.
## Using Dockge
### First Login
Once installation finishes, click the Web UI button, or browse to `http://TRUENAS_IP:5001`, where TRUENAS_IP represents the IP address of your NAS.  You'll be prompted to create a username and password for this instance.
### Your first stack
Now let's create a stack, and for demonstration purposes we'll use Jellyfin.  First, create a dataset for its metadata, configuration, and such, under `docker/data`.

Then, in Dockge, click **+ Compose**.  This will take you to the main page to enter and edit your stacks.  The top-right text field is for the Compose YAML, and is prepopulated with a basic Nginx stack; the text field below it is for the `.env` file if used.  Empty the top field, and paste this in its place:
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: 568:568
    volumes:
      - /mnt/software/docker/data/jellyfin/data:/config
      - /mnt/software/docker/data/jellyfin/cache:/cache
    restart: unless-stopped
    environment:
      - JELLYFIN_PublishedServerUrl=http://TRUENAS_IP:8096
    ports:
	  - 8096:8096
```
Note that the volumes above are storing data in the `docker/data/jellyfin` dataset created above.  You'd also want to add paths for any media.  Once you've finished, click **Deploy.**
## References
This guide closely follows the method described in https://www.youtube.com/watch?v=R0Vdj1culo0.