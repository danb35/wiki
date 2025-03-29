---
title: Let's Encrypt Certificate for TrueNAS CORE
description: Using TrueNAS CORE's built-in capacity to obtain a certificate from Let's Encrypt
published: true
date: 2025-03-29T16:26:29.410Z
tags: 
editor: markdown
dateCreated: 2025-03-29T16:26:29.410Z
---

# Let's Encrypt Certificate for TrueNAS CORE
## Introduction
TrueNAS SCORE include the ability obtain a trusted TLS certificate from Let's Encrypt and keep it up to date automatically.  If you meet the requirements below, this is probably the easiest way to keep an up-to-date, trusted certificate for your NAS.
## Prerequisites
* You must own or control a real Internet domain, which will incur a small cost--they start around US$10 per year from registrars like [namecheap.com](https://www.namecheap.com/) or [Cloudflare](https://www.cloudflare.com/).
* DNS for this domain must be hosted by [Route53](https://aws.amazon.com/route53/).

If you own a domain but host your DNS with a different provider, and are unable or unwilling to change to one of these, you may instead want to consider using `acme.sh` or some other client to obtain a certificate, and deploying that certificate to your NAS following [this guide](/fester/maintain-truenas/script-cert-core).
## Instructions
### Create an ACME DNS-Authenticator
In the TrueNAS Web UI, browse to System -> ACME DNS.
![core-acme-dns.png](/core-acme-dns.png)
Next to ACME DNS-Authenticators, click **Add.**
![core-acme-dns-add.png](/core-acme-dns-add.png)
Enter any name you like, select Route53 (the only option) as the authenticator, and enter your credentials.

Click **Submit.**
### Create a Certificate Signing Request (CSR)
Next, create a CSR.  Browse to System -> Certificates, and click **Add.**

Set **Type** to `Certificate Signing Request`.  In **Subject Alternate Names,** enter the domain names you want to be covered by this certificate.  You can fill in the remaining fields without a default with whatever you like.
![core-csr-add.png](/core-csr-add.png)
Click **Submit.**
### Request the certificate
Now we can request the certificate itself.  Click on the kebab menu to the right of your newly-created CSR, and select **Create ACME Certificate.**
![core-acme-request.png](/core-acme-request.png)
Fill in the form.  Identifier can be anything you like.  Check the box to accept the terms of service for your CA (Let's Encrypt's current TOS, as of February 2025, are [here](https://letsencrypt.org/documents/LE-SA-v1.5-February-24-2025.pdf)).  **Set Renew Certificate Days to 30.**  This field controls when (i.e., how many days before expiration) TrueNAS begin trying to renew the certificate; Let's Encrypt recommends that renewal attempts begin when 1/3 of the certificate lifetime remains, which would mean 30 days.  The default for this field is a nonsensical 10 days; iXSystems have had tickets for this issue for years and have yet to correct it.
![core-acme-cert.png](/core-acme-cert.png)
Set **ACME Server Directory URI** to `https://acme-v02.api.letsencrypt.org/directory` (i.e., the option without `-staging` in the URI).

Under **Domains,** for each domain you've entered in the **Subject Alternative Names** field in the CSR, choose the ACME DNS authenticator you created above.  Then click **Submit.**  Your certificate will be requested and, if successful, renewed automatically.
### Configure your system
Now that you've created the certificate, you'll need to configure your NAS to use it.  Browse to System -> General, and set **GUI SSL Certificate** to the one you just created.  Then click **Save.**

If you've configured any other services to use a certificate (FTP and WebDAV would be the most likely candidates), set them to use the newly-created certificate as well.  Browse to Services, click the pencil next to the service in question, and set the correct certificate.