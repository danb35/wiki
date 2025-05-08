---
title: Scripted Certificate Import for SCALE
description: Automating TLS certificate deployment to TrueNAS SCALE
published: true
date: 2025-05-08T13:07:44.440Z
tags: letsencrypt
editor: markdown
dateCreated: 2025-03-28T22:12:10.570Z
---

# Scripted Certificate Import for SCALE
## Introduction
If TrueNAS' built-in ability to obtain and renew a certificate from Let's Encrypt fails to meet your needs for whatever reason, the TrueNAS API still allows deployment of a new certificate to be automated using [a script](https://github.com/danb35/deploy-freenas).  This script need not reside on your NAS, and need not be used with Let's Encrypt certificates specifically, but the illustration below will run on the NAS.
## Prerequisites
The script discussed in this guide requires Python 3, its OpenSSL module, and the TrueNAS API client.  All these are installed by default in TrueNAS SCALE 24.10 and later.  If you're running this script on a remote system, the [README](https://github.com/danb35/deploy-freenas/blob/master/README_truenas.md) discusses the installation process of the dependencies into a standard Linux environment.

Other than for the script itself, we'll be generating a certificate from Let's Encrypt, so you'll need to own or control a public Internet domain.  We'll be using [acme.sh](https://acme.sh) with DNS validation, so DNS for your domain will need to be hosted with one of the 150 or so providers it supports.
## Instructions
### Install acme.sh
Recent versions of TrueNAS prohibit executing code from a user's home directory.  As a result, you'll need to create a location on your pool to store acme.sh and other scripts.  For purposes of this guide, this location will be called `/mnt/tank/scripts`.  To install acme.sh there, run the following commands:
```text
cd /mnt/tank/scripts
git clone --depth 1 https://github.com/acmesh-official/acme.sh.git
cd acme.sh
./acme.sh --install  \
--home /mnt/tank/scripts/acmesh \
--accountemail  "my@example.com"
```
Replace `my@example.com` with your actual email address, so that Let's Encrypt can send you emails if needed.

Now set acme.sh to use Let's Encrypt by default:
```text
cd /mnt/tank/scripts/acmesh
./acme.sh --set-default-ca --server letsencrypt
```

Finally, configure a cron job.  In the TrueNAS web UI, go to System -> Advanced Settings, and click **Add** for a Cron job.  Enter whatever you like for the description.  For the command, enter `/mnt/tank/scripts/acmesh/acme.sh --cron`.  Run it as root daily.  For best reliability, set it to run at a custom time other than at midnight every day.
### Issue the certificate
These instructions will assume you're using Cloudflare for your DNS.  If you're using a different DNS provider, consult the acme.sh docs [here](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) and [here](https://github.com/acmesh-official/acme.sh/wiki/dnsapi2) for specifics.  You'll also need an API token from Cloudflare with permissions of Zone / Zone / Read and Zone / DNS / Edit for your domain.  Then run these commands:
```text
cd /mnt/tank/scripts/acmesh
export CF_Token=(your token)
./acme.sh --issue --dns dns_cf -d truenas.yourdomain
```
Watch the output.  If it reports success, congratulations!  Make a note of the paths to the private key and the "Full chain" certificate.
### Install and configure the deploy script
You'll next download the deploy script.  Run these commands:
```text
mkdir /mnt/tank/scripts/deploy-truenas
cd /mnt/tank/scripts/deploy-truenas
wget https://github.com/danb35/deploy-freenas/raw/refs/heads/master/deploy_truenas.py
chmod +x deploy_truenas.py
```

Create a file called `deploy_config` using your favorite text editor.  Its contents should look like this:
```python
[deploy]
api_key = YourReallySecureAPIKey
privkey_path = /some/other/path
fullchain_path = /some/other/other/path
connect_host = truenas.yourdomain
protocol = wss
verify_ssl = false
delete_old_certs = true
```
The `verify_ssl = false` setting will cause the script to ignore any certificate errors, such as an expired or invalid certificate.  Once you've imported a trusted certificate, it's recommended to set this to `true`.

The `delete_old_certs` setting will cause the script to delete any certificates in your TrueNAS system whose name begins with `letsencrypt`, other than the one you're currently importing.  It's recommended to set this to `true` to avoid the list of certificates in the UI being too cluttered.
### Deploy the certificate
Now that the deploy script is installed and configured, tell acme.sh to run that script when a new certificate is obtained.  To do that, run `/mnt/tank/scripts/acmesh/acme.sh --install-cert -d truenas.yourdomain --reloadcmd /mnt/tank/scripts/deploy-truenas/deploy_truenas.py`.  This will deploy the certificate to your NAS.  The cron job set up above will run `acme.sh` daily and renew the certificate when it's within 30 days of expiration.  When it renews the certificate, it will run the deploy script to import it into your TrueNAS system.
## Conclusion
This guide has demonstrated use of the deploy script in connection with `acme.sh`.  However, it can be used with any certificate, however obtained, and from whatever certificate authority.