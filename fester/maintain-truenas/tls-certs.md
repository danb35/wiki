---
title: TLS Certificates on TrueNAS
description: How to create and/or deploy them
published: true
date: 2025-03-28T22:30:09.491Z
tags: 
editor: markdown
dateCreated: 2025-03-28T22:30:07.661Z
---

# TLS Certificates on TrueNAS
## Introduction
Modern web browsers will complain about insecure connections when you're accessing a web site, even an internal one, without using HTTPS.  Using HTTPS requires a certificate, and TrueNAS includes several methods to handle them.
## Self-signed certificate
TrueNAS creates a self-signed certificate on installation, which is valid for about one year in recent versions of TrueNAS.  This certificate will allow encryption of traffic for the web interface and certain other services, but all browsers will give warnings for a self-signed certificate.  When this certificate nears expiration or has expired, the web interface will give you a warning.

To replace the self-signed certificate with a new one, see the following instructions:
* TrueNAS CORE
* TrueNAS SCALE
## Obtain a Let's Encrypt certificate
Recent versions of TrueNAS CORE, and every version of TrueNAS SCALE, include limited functionality to obtain a trusted certificate from Let's Encrypt and renew it automatically.  This functionality requires that DNS for your domain be hosted buy a compatible provider: Route53 for CORE; or Cloudflare, OVH, or Route53 for SCALE.  Instructions for this process are below:
* TrueNAS CORE
* TrueNAS SCALE
## Import a certificate automatically
If DNS for your domain isn't hosted by a compatible provider, or you'd otherwise prefer to manage your certificates (whether from Let's Encrypt or elsewhere) on a different system, you can use a script to automate importing a certificate.  Instructions for this process are below:
* [TrueNAS CORE](/fester/maintain-truenas/script-cert-core)
* [TrueNAS SCALE](/fester/maintain-truenas/script-cert-scale)
