---
title: 7.1 Apps on TrueNAS SCALE
description: Current status and recommendations
published: true
date: 2025-05-08T13:11:17.441Z
tags: apps
editor: markdown
dateCreated: 2024-06-01T15:35:57.319Z
---

# Apps on TrueNAS SCALE
## History
iXSystems have included "plugins" as a feature in FreeNAS, and then TrueNAS CORE, since FreeNAS 8.0, up through the current release of TrueNAS CORE 13.0.  They've always been troublesome, largely because they've been difficult to keep up to date.  On 25 April 2023, iXSystems [announced](https://www.truenas.com/blog/the-future-of-truenas-plugins-is-apps/) that plugins on TrueNAS CORE would be deprecated, and that the way ahead for a plugin-like experience would be the apps ecosystem on TrueNAS SCALE.

Apps were one of the headline features of TrueNAS SCALE when it was first announced.  From its first release (code-named Angelfish) through the current release (Dragonfish), the apps ecosystem has been based on Kubernetes.  In addition to the hundred or so apps provided by iXSystems between their "official" and "community" repositories, [TrueCharts](https://truecharts.org/) developed a third-party app catalog, currently containing nearly 800 apps.  In this author's opinion, the TrueCharts catalog, and the apps in it, is far superior to that provided by iXSystems, in terms of quantity, consistency, and configurable features.

## Current Status
On 29 May 2024, iXSystems [announced](https://forums.truenas.com/t/the-future-of-electric-eel-and-apps/5409) a drastic change to their apps ecosystem that will take place with the release of SCALE 24.10 (Electric Eel) scheduled for October 2024.  Kubernetes will be going away, and the apps will instead be based on Docker Compose.  Their announcement promised that they would create an equivalent app catalog, and that the upgrade to Electric Eel would seamlessly migrate their apps to the new system.

TrueCharts [responded](https://truecharts.org/news/scale-deprecation) that their apps are entirely dependent on Kubernetes, they would not move to Docker Compose, and that they would develop a migration strategy to a supported platform.  They have deprecated SCALE as a supported platform and will be making no further updates to its app catalog.  In early July 2024, they removed their SCALE apps catalog entirely.  They plan a beta release of their migration strategy by 1 July 2024.

## Advice
At this point, it's hard to recommend installing any apps on SCALE, especially on a new system.  If you have an immediate need for apps in a point-and-click sense, the "official" catalog is the only option currently available.  Another option would be to create a sandbox using Jailmaker, into which you can optionally install Docker and whatever web interface you like.  Instructions can be found in this video:
https://www.youtube.com/watch?v=S0nTRvAHAP8

