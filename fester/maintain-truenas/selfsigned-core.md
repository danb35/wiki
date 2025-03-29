---
title: Self-signed Certificate for TrueNAS CORE
description: How to create a self-signed certificate in TrueNAS CORE
published: true
date: 2025-03-29T14:17:14.958Z
tags: 
editor: markdown
dateCreated: 2025-03-29T14:17:14.958Z
---

# Self-signed Certificate for TrueNAS CORE
## Introduction
TrueNAS creates a self-signed TLS certificate on installation, which is valid for about a year.  Like any certificate, this will enable secure communication over HTTPS between a client system and your NAS.  However, it will not be trusted by browsers, so you'll get certificate errors when you browse to your NAS via HTTPS.

When this certificate nears expiration, you'll begin to get alerts in the web interface (and by e-mail, if you have that configured).  You can ignore them (you're already getting certificate errors if you're using a self-signed certificate), but the way to correct them is to generate a new self-signed certificate.
## Instructions
### Create a certificate authority
In the TrueNAS web interface, browse to System -> CAs, and click **Add.**  You can leave the default values as they are, and fill in the mandatory fields as desired.  Once you've finished, click **Submit.**
![core-create-ca.png](/core-create-ca.png)
Once you've created the certificate authority, click the kebab menu to the right, and choose **Export Certificate.**  You can then download this certificate and import it into your browser or operating system's trust store, which will allow your browser to trust the certificate you're about to create.
### Create a certificate
Then go to System -> Certificates and click **Add.**  The Type should default to Internal Certificate, which is correct.  Set **Signing Certificate Authority** to the CA you just created.  Fill in the other mandatory fields as desired.  Note that **Subject Alternate Names** needs to include every address you use to access your NAS's web UI (and FTP and WebDAV, if you're using those services), whether that's an IP address, a hostname, a fully-qualified domain name, or some combination of those.  There's no practical limit to the number of SANs you can include.  Once you've finished, click **Submit.**
![core-create-cert.png](/core-create-cert.png)
### Configure your system
Now that you've created the certificate, you'll need to configure your NAS to use it.  Browse to System -> General, and set **GUI SSL Certificate** to the one you just created.  Then click **Save.**

If you've configured any other services to use a certificate (FTP and WebDAV would be the most likely candidates), set them to use the newly-created certificate as well.  Browse to Services, click the pencil next to the service in question, and set the correct certificate.
### Delete the old certificate
The system will continue to alert you to the expired certificate until you delete it.  To do so, browse to System -> Certificates, click the kebab menu to the right of the old certificate, and select **Delete.**
