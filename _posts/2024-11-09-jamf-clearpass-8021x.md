---
layout: single
classes: wide
title: "802.1X Device Authentication with Jamf Pro and HPE Aruba ClearPass"
header:
  teaser: assets/images/default-image.jpeg
excerpt: "Implementing 802.1X device authentication in Jamf Pro with HPE Aruba ClearPass"
date: 2024-11-09 15:25:00 -0500
categories:
  - Career
  - Mac Admin
tags:
  - jamf
  - 802.1X
  - apple
  - mac
  - networking
toc: true
toc_sticky: true
permalink: 8021x-device-auth-with-jamf-and-clearpass/
---

This post is about how The Ohio State University implemented 802.1X device authentication in Jamf Pro with HPE Aruba ClearPass. I briefly mentioned this in my [JNUC session](https://reg.jnuc.jamf.com/flow/jamf/contentlibrary/sessioncatalogcontentlibrary/page/page/session/1728584964589001nrc0){:target="\_blank"} about how Ohio State utilizes Jamf Pro for device management. My slides from this session can be found in [this repository](https://github.com/raydemay/jnuc2024){:target="\_blank"}.

Some work went into this that I did not have a lot of involvement in, so there will be some things that I can't be specific on. However, I can provide a high-level overview of the process and some key points to consider. I mainly wanted post this to talk about the way we set this up to not need MAC addresses for 802.1X authentication now that MacOS 15 includes [MAC address randomization](https://support.apple.com/guide/security/wi-fi-privacy-secb9cb3140c/web){:target="\_blank"}.

# Highlights

- Jamf AD CS Connector for client certificate requests
- HPE Aruba ClearPass Jamf Extension
- Single configuration profile for wired and wireless
  - Profile includes the certificate payload and network settings
  - All mobile devices also have a wireless profile
  - Wireless is configured for [eduroam](https://internet2.edu/eduroam-global-roaming-access-service/){:target="\_blank"}

Here is a 30,000ft view of how these things work together to enable 802.1X authentication:

1. Device enrollment installs configuration profile and triggers webhook
2. Webhook sends JSSID to ClearPass to initiate API lookup
3. Extension makes API call to get device inventory
4. Device authenticates with certificate
5. Certificate is validated by ClearPass
6. ClearPass authenticates and authorizes device

# Configuration

There is a decent amount of work that occurs outside of Jamf Pro. In fact, the Jamf side was probably the most straightforward part for me (considering I am the Jamf platform owner for the university, I guess that makes sense). The bulk of the work happens in ClearPass, with some AD and Windows Server work included. I have a link to relevant instructions provided by the vendor when it is important, rather than trying to summarize them in this post.

The first thing to figure out is how to request and deliver cetificates to the devices. We chose to use the Jamf AD CS Connector for client certificate requests.

## Jamf AD CS Connector

As far as I'm concerned, PKI is witchcraft. I can't tell you how our implementation works in any detail, just that I was able to install the Jamf AD CS connector onto a VM that was built for me based on the [instructions provided by Jamf](https://learn.jamf.com/en-US/bundle/technical-paper-integrating-ad-cs-current/page/Overview_ADCS.html){:target="\_blank"}. The certificate template was created by our Active Ddirectory team for this use case.

<figure>
  <img src="https://learn-be.jamf.com/bundle/technical-paper-integrating-ad-cs-current/page/images/AD_CS_Jamf_Cloud_Reverse_Proxy_Communication.png?_LANG=enus" alt="Jamf AD CS Connector">
  <figcaption>Jamf AD CS Connector with Reverse Proxy or Load Balancer (Credit: Jamf)</figcaption>
</figure>

Once the connector application is up and running, the next step is to configure AD CS connector in Jamf for client certificate requests. This just requires adding a certificate authority under Settings > Global > PKI Certificates. Jamf's documentation that is linked above covers this configuration as well. Once you have the CA configured, you can use it to request certificates from your AD CS server in a configuration profile.

The next step is to configure the network to allow the use of these certs.

## ClearPass Jamf Extension

This setup has a lot of parts. Here is the documentation straight from HP:

[HPE Aruba ClearPass Jamf Extension Documentation](https://support.hpe.com/hpesc/public/docDisplay?docId=a00126745en_us){:target="\_blank"}

There is information for both Jamf admins and network admins to read in the doc. The important things for a Jamf admin are to generate API credentials for the extension, and for the network admin configuring the extension to request a Skyhook URL. This enables Jamf to send JSON to ClearPass to register devices in its Endpoints database.

I created a diagram of my understanding of how the extension communicates with Jamf. The diagram below shows the communication flow between ClearPass and Jamf, with the Skyhook service being the intermediate step for the webhook.

The basic steps are as follows:

1. Jamf sends a webhook to the Skyhook service with a JSSID of a device.
2. The service forwards that information to our ClearPass infrastucture.
3. ClearPass then communicates back to Jamf using the API to get more information about the device.

<figure>
  <img src="{{site.url}}/assets/images/ArubaJamfExtension.drawio.png" alt="Aruba Jamf Extension Skyhook">
  <figcaption>Aruba ClearPass Jamf Extension Skyhook Flow</figcaption>
</figure>

The SQL query that enables authentication and authorization without MAC Address is as follows:

```sql
select te.attributes->>'JAMF Serial Number' as Serial_Number from tips_endpoints as te where te.attributes->>'JAMF Serial Number' = '%{Certificate:Subject-AltName-DNS}'
```

I'm not 100% sure if there are values that are custom to our ClearPass implementation here. The documentation has all of the attributes listed as an appendix, so it shouldn't take a ton of work to get an implementation of this. There are other attributes that could be used in this query to only allow managed or supervised devices on the network, so play with this query and see what works best for your environment. Adding too many things to this query can be a performance hit, so it's important to keep it as simple as possible.

## Webhooks

Four webhooks are configured in Jamf to handle different events:

- ComputerEnrolled
- ComputerInventoryCompleted
- MobileDeviceEnrolled
- MobileDeviceInventoryCompleted

The webhook settings needed for this are included in the Jamf Extension documentation. The important piece is the unique URL provided by HPE for their Skyhook service.

<figure>
  <img src="{{site.url}}/assets/images/jamf-webhook-config.png" alt="example Jamf webhook settings">
  <figcaption>Example Jamf webhook settings</figcaption>
</figure>

The webhooks are the primary way to make ClearPass aware of devices, but there is also a way to query Jamf to get every device in the instance. We have this lookup run twice per day as a way to detect changes and removals that the webhooks aren't designed to handle. This is much more resource intensive, so it's done early in the morning and later in the evening. Before this extension was created, using the API to just get every device in Jamf was the only way to do this.

# Profile Configuration and Deployment

## Configuration Profile

Since Apple requires both the network payload and certificate payload in the same profile, I created one profile that has both the wired and wireless 802.1X settings and the certiicate payload. I am going to talk mainly about the macOS profile, but the mobile device settings are basically the same, except for the wired configuration.

### Certificate Payload

The certificate payload includes device cert request and trust chain for both AD CA and RADIUS server certificates. Only the root certificate for the network server is included, because there is an option to trust authentication servers by name. See Apple's [documentation on 802.1X](https://support.apple.com/guide/deployment/connect-to-8021x-networks-depabc994b84/web){:target="\_blank"} for more details. The client certificate is configured as follows:

- Certificate CN is **$SERIALNUMBER@FQDN**

  $SERIALNUMBER is the Jamf variable for the device serial. A special FQDN was created that indicates that the certificate came from Jamf. Our FQDN ends in osu.edu so that it can be used for eduroam authentication at other instituitions. This is why the AD CS Connector instructions state to set **Supply in the request** in the Subject Name settings tab of the template.

- Certificate SAN is **$SERIALNUMBER**

  This allows the certificate to be used to authenticate _without_ the MAC address needing to be known. This was done before macOS 15 adding MAC Address Randomization, which was a happy accident. We chose DNS name, even though it most certainly is _not_ a valid DNS name, just because the SAN field only has four pre-defined types in Jamf Pro and none of them really made sense over the other, and was simple to implement in the ClearPass SQL query.

<figure>
  <img src="{{site.url}}/assets/images/jamf-clientcert-settings.png" alt="Jamf client certificate settings">
  <figcaption>Client certificate settings in Jamf Certificate payload</figcaption>
</figure>

We set the "Allow all apps access" option to also use this certificate for Cisco Secure Client VPN authentication. This only works for macOS, as the iOS/iPadOS version of the client requires the VPN payload and certificiate to be included in the same MDM profile, as it uses its own certiicate store.

### Network Payload

The network payload includes all settings for 802.1X EAP-TLS authentication for both wireless and wired networks. The payload settings should be straightforward once everything else is in place. Here are some screenshots of the settings for the wireless payload:

<figure>
  <img src="{{site.url}}/assets/images/jamf-wirelessnetwork-payload.png" alt="Jamf Wireless network payload WiFi settings">
  <figcaption>WiFi network payload settings</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/assets/images/8021x-settings-1.png" alt="Network payload Protocols setting">
  <figcaption>Network protocol settings</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/assets/images/8021x-settings-2.png" alt="Network payload Trust setting">
  <figcaption>Network trust settings</figcaption>
</figure>

Duplicate these settings for a second network payload, selecting **Any Ethernet** as the network interface. This was the option for wired that worked best for us in our testing because it handled dongles and docking stations.

## Deployment

The easiest thing to do is to just scope this to all devices, but if you are replacing password authentication with device certificate authentication to the same SSID, there is an issue. Scoping the 802.1X payload, then removing the old payload, will cause the device to forget the network and require the user to do some manual work to get connected. The easiest workaround is to create a second SSID with PSK authentication as a temporary network to handle the swap. Once all devices have connected to this new network, you can remove the old profile and deploy the new 802.1X profile. Our deployment was a bit easier, because we were switching from one network to another.

# Extras

When we started to plan this project, one resource that I used a ton was the #8021x channel of [Mac Admins Slack](https://macadmins.org), and one of the things that I came across were some scripts to handle situations where macOS would fail over to a user level profile instead of the system one that the MDM profile created. Here is ther script for the WiFi identity (one also exists for Ethernet):

[https://github.com/eth-its/autopkg-mac-recipes-yaml/blob/main/Scripts_Tools/8021X-wifi-identity.sh](https://github.com/eth-its/autopkg-mac-recipes-yaml/blob/main/Scripts_Tools/8021X-wifi-identity.sh)

The logs will tell you if you are seeing behavior that requires this script.

```bash
log stream --predicate 'subsystem == "com.apple.eapol"' --info --debug
```

I also wanted to mention the strategy we use for network authorization. We put devices in what we decided to call Network Roles. We utilize an extension attribute that puts device in a smart group which is reported to ClearPass to assign network access. The extension attribute is set on devices based on a combination of properties in the devices's record in our ServiceNow CMDB. Devices can authenticate without this role, but it is necessary to be able to access certain resources. A device needs to do an inventory update after network role is set to trigger webhook, otherwise ClearPass will not be aware of the role until the full sync occurs. ClearPass is looking at the inventory of the device to see that it is a member of one of the role groups. I left this out of the SQL query because I wanted to focus on authz and not the extra stuff that we have.
