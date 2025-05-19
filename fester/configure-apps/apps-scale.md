---
title: 7.1 Apps on TrueNAS Community Edition (SCALE)
description: Current status and recommendations
published: true
date: 2025-05-19T15:30:47.879Z
tags: apps
editor: markdown
dateCreated: 2024-06-01T15:35:57.319Z
---

# Apps on TrueNAS Community Edition (SCALE)

TrueNAS, and FreeNAS before it, have supported user-installed software, using a variety of mechanisms (and often more than one mechanism at the same time), since the release of FreeNAS 8.0 in 2010.  With the release of TrueNAS 24.10 in October 2024, iXSystems removed the Kubernetes-based apps system that had been used from 22.12 to 24.04, and replaced it with a Docker-based system.  TrueNAS 25.04 added experimental support for Linux Containers (LXCs), and also supports virtualization.  Thus, users have several options to run additional software on their TrueNAS systems.

# Options
Users have five options to install additional software on their TrueNAS systems:
* Install the software directly on the NAS
* Use the built-in Apps functionality
* Use Docker Compose "Custom apps," with or without the assistance of software like Portainer or Dockge
* Install the software in a LXC
* Install the software in a Virtual Machine

## Install directly on the NAS
TrueNAS is an increasingly locked-down environment, and users ordinarily should not install software directly in that environment.  However, this may be appropriate for scripts, or occasionally for statically-linked binaries (such as those typically produced by Go), which can just be dropped in a convenient directory and run from there.

Note that in current releases of TrueNAS, the `/home` directory has code execution disabled.  As a result, any user-installed software must reside elsewhere.
## Use the built-in Apps functionality
This is the "easy button."  If the software you want to use is available as an app (just under 200 as of this writing in May 2025), you can point, click, perhaps fill in a few fields in a form, and install it.

With this ease, however, comes a loss of flexibility.  Many decisions have been made for the user, perhaps as the user would not prefer them to have been made, for the sake of ease of installation and use.

Moreover, this system depends on iXSystems' ongoing maintenance.  Historically, this has been uneven at best.  TrueNAS is on its fourth apps/plugins ecosystem; iXSystems having discarded three prior systems after deciding they were too much work to maintain.  In this author's opinion, iXSystems cannot be relied on to maintain and support this catalog.
## Custom Docker Compose apps
Users who need software that isn't in the apps catalog, need options that aren't available in those apps, don't trust iXSystems to continue to maintain the apps catalog, or just otherwise feel like it, can run software using Docker.  This requires some knowledge on the user's part--while a great deal of software publishes Docker images and Compose files, these will often need modification and/or customization to match the configuration and requirements of the user's environment.  Such customization will generally be beyond the scope of this guide or the TrueNAS documentation.  There are in turn several ways to do this.
### "Custom app" through the TrueNAS GUI
TrueNAS will let you run arbitrary Docker containers using the "Custom App" form.  Fill in the fields on that form appropriately for the application you want to use, and it will run that application.
### "Install via YAML" through the TrueNAS GUI
TrueNAS will also let you enter a Compose file through its GUI.  This allows for complex, multi-container applications that would not be possible using the "Custom App" form.  This method does not directly allow for use of a `.env` file (commonly used to store environment variables for the stack), but this limitation can be worked around by including files stored elsewhere on the system.  The UI for this isn't as clean or featureful as with using software like Portainer or Dockge which is designed for this purpose, but it's nonetheless possible without installing any additional software.
### Docker at the CLI
The normal Docker CLI tools are now part of TrueNAS, so you don't need to use a GUI at all if you'd rather do it caveman-style.  `docker compose up -d` works the same way as on a regular Linux system.
### Dockge/Portainer
Probably the most common way for users to run custom Docker apps is to use [Dockge](https://dockge.kuma.pet/) or [Portainer](https://www.portainer.io/).  Both are available as TrueNAS apps, and provide a featureful GUI to manage your Docker applications.  You'd install Dockge or Portainer using the TrueNAS app, and then use that to install and manage your other Docker applications
## Install in a LXC
Beginning in TrueNAS 25.04, TrueNAS provides experimental support for Linux Containers.  These in many cases provide a lightweight alternative to virtual machines, though only for Linux-based operating systems.  These are considered an advanced feature and may not be documented here.
## Install in a VM
TrueNAS has long supported virtualization, allowing users to run virtual machines to run virtually any desired operating system and software.  This, again, is considered an advanced feature which may not be documented here.