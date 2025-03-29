---
title: Nginx Proxy Manager
description: Installing and Configuring Nginx Proxy Manager in TrueNAS SCALE Electric Eel (24.10)
published: true
date: 2024-12-30T23:07:28.881Z
tags: 
editor: markdown
dateCreated: 2024-11-10T12:29:03.837Z
---

# Nginx Proxy Manager
## Introduction
Nginx Proxy Manager (NPM) provides a web interface to configure the popular web server Nginx as a reverse proxy.  It can be used for many purposes, but this guide will describe using it to provide HTTPS/TLS termination for other applications running on your TrueNAS system.
## Prerequisites
* You must own or control an actual public domain.  That domain doesn't need to be accessible from the public Internet, but it must have public-facing DNS records.  This guide will use `example.com` as a placeholder for this domain.
* You **do not** need to open or forward any ports on your router, or otherwise make anything on your LAN accessible to the Internet.  You can if you choose, but that is outside the scope of this guide.
* To follow this guide exactly, you must host your domain's DNS at Cloudflare.  Cloudflare provides free DNS hosting, among many other services (not all of which are free).
	* Your DNS provider matters because we're going to be obtaining certificates from Let's Encrypt, and that requires they validate your control over your domain.  In order to do that without exposing your apps to the public, NPM will need to be able to make updates to your DNS records.
	* NPM supports many other DNS hosts, and you should be able to adapt this guide to them, but this guide will be using Cloudflare.
* You've generated an [API Token](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) from Cloudflare that you'll use to allow NPM to make these changes.
  * This token must have permissions of `Zone:Zone:Read` and `Zone:DNS:Edit`.
* You can control your **local** DNS, making custom entries to point names of your choosing toward IP addresses on your LAN.
	* You can use router software like OPNsense or pfSense for this, use a local DNS server like Pi-Hole, or any other means.
	* This is required because the certificate you're going to generate will cover **names,** not IP addresses.  Browsing by IP (even if it were possible) would give you certificate errors.
  * This is also required because NPM will use the requested hostname to determine the proxy target.  Browsing to the IP address will just give you the default NPM welcome page, not the app you're looking for.
  * As a distinctly suboptimal alternative, you can put entries pointing to your LAN devices in public DNS.
* You're running TrueNAS SCALE Electric Eel (24.10).  This version completely changed how apps work, so this guide will not be applicable to any earlier version of TrueNAS.
* This guide will be using the point-and-click apps ecosystem in TrueNAS SCALE.  There are many other ways to configure NPM, including custom Compose apps, that are outside the scope of this guide.
## Preparation
NPM will need to listen on ports 80 and 443 of your system, and by default the TrueNAS web interface already does that.  You'll therefore need to change this.  In the TrueNAS web interface, go to System (1) -> General Settings (2), then click Settings in the GUI section (3).
![npm-guisettings1.png](/npm-guisettings1.png)

Change HTTP port to 81 (1), and HTTPS port to 444 (2), then click Save.
![npm-guisettings2.png](/npm-guisettings2.png)

Confirm the restart of the web GUI.

At this point you'll probably need to log back into the web interface.  Browse to `http://ip_of_truenas:81` and do so.
## Install NPM
In the TrueNAS web interface, browse to Apps, then click Discover Apps.  Find `Nginx Proxy Manager` and click Install.  Set the HTTP port to 80, and the HTTPS port to 443.  You can also change the WebUI Port if desired, but the remainder of this guide assumes you've left it at the default.
![npm-network.png](/npm-network.png)

Then click Install.  Let the application deploy, which may take a few moments.
## Configure NPM
### DNS preparation
Create a **local** DNS record pointing `npm.lan.example.com` to the IP address of your NAS.
### Log in
To access the NPM UI, browse to `http://ip_of_truenas:30020`.  Log in with an email address of `admin@example.com` and a password of `changeme`.  You'll be prompted to change these once you log in.
### Create a certificate
For this step, you'll need your domain name and the API token.  It's recommended that you reserve a subdomain for your LAN resources, e.g., if your domain is `example.com`, you'd use `lan.example.com` for your local domain.

In the NPM UI, go to **SSL Certificates,** and click **Add SSL Certificate.**  For domain names, enter both `lan.example.com` and `*.lan.example.com`.  Enter your email address--Let's Encrypt will use this to notify you if your certificate is about to expire.  Turn on **Use a DNS Challenge.**

The form will change at this point.  Choose **Cloudflare** as your DNS provider.  When you do, **Credentials File Content** will change to look like this:
```text
# Cloudflare API token
dns_cloudflare_api_token=0123456789abcdef0123456789abcdef01234567
```
Paste in your API token after the `=` (replacing `0123456789...`), leaving everything else the same.

The form should look like this:
![npm-addcert.png](/npm-addcert.png)

Then click to accept the Let's Encrypt Terms of Service, and click Save.  Creation of the certificate will take a few moments, and you'll be returned to the SSL Certificates page.
### Create a host
Now that you've created a wildcard certificate, let's create a proxy host for one of your apps.  Still in the NPM UI, go to Hosts -> Proxy Hosts and click on **Add Proxy Host.**

On the Details tab, start by entering the FQDN you'll be using (e.g., if you're setting up a proxy for Sonarr, you might use `sonarr.lan.example.com`).  You'll leave scheme set to http, and for IP you'll enter the IP of your NAS.  In Port, you'll enter the port for that app (30027 by default for Sonarr).  Turn on websockets support and Block Common Exploits.
![npm-host1.png](/npm-host1.png)

Then go to the SSL tab.  Under SSL Certificate, select the certificate you created above.  Click on Force SSL and Enable HTTP/2 support.  HSTS is not recommended at this time.[^1]
[^1]: [HTTP Strict Transport Security (HSTS)](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) is a security policy that requires clients, once they connect to your site/service, to only connect via HTTPS with a valid certificate for a certain time, ordinarily a year.  If there's any issue with your TLS configuration--most likely that certificates fail to renew--this can lock you out of your services.  Once you're sure everything's working properly and your certificates are renewing as they should, you can come back and enable HSTS if you choose.

![npm-host2.png](/npm-host2.png)

Click Save.

Finally, update your local DNS records to add this hostname as an alias of `npm.lan.example.com`.
### Test
Browse to the host you just created.  You should see the page for that app, and your browser should indicate that HTTPS is enabled with a trusted certificate.
### Your NAS
We changed the listening ports for your TrueNAS installation to 81 and 444 above.  To restore the ability to reach your NAS on the standard ports, you can add it as another host in NPM, as described above.  You'd direct the host to port 81 and use a name like `nas.lan.example.com`.
### Repeat
Repeat the two steps above for any additional applications.
## Other Options
This guide has created a single wildcard certificate, in the expectation that any of your LAN services will use subdomains of `lan.example.com`.  If that isn't the case for you, or if you'd just prefer to create individual certs for each service, you can dispense with creating the wildcard cert above, and instead have NPM create a new cert for each proxy host.  Be aware of the [Let's Encrypt rate limits](https://letsencrypt.org/docs/rate-limits/) should you decide to do this.
## Conclusion
This guide has described installation and configuration of Nginx Proxy Manager to allow TLS termination for the apps on your TrueNAS server.  Consult the [Nginx Proxy Manager documentation](https://nginxproxymanager.com/guide/) for more details on its configuration.

Many guides configure your network in such a way that the underlying apps are not directly accessible via their ports (i.e., for the app above, via port 30027).  This guide **does not** implement such a configuration due to limitations in the apps ecosystem.  As a result, you or other users on your network will be able to directly access the apps, without TLS, and without going through your proxy, unless you take other measures to restrict this.  Those measures are outside the scope of this guide.
