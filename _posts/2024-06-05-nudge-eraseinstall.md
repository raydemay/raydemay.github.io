---
layout: single
title: "Using Nudge and erase-install to upgrade MacOS"
header:
  teaser: assets/images/nudge-major-upgrade.png
excerpt: "Nudge and erase-install can combine for a powerful MacOS upgrade workflow"
date: 2024-06-05 21:30:00 -0400
categories:
  - Mac Admin
tags:
  - jamf
  - nudge
  - erase-install
  - apple
  - mac
  - updates
---

A common challenge with managing MacOS updates is controlling when a major upgrade can be done. There are plenty of articles out there on how to set software update restrictions to block a major release for up to 90 days. Instead, I want to explain how I use [Nudge](https://github.com/macadmins/nudge) and [erase-install](https://github.com/grahampugh/erase-install) in Jamf Pro to get Macs all upgraded to the newest release, treating it similar to a minor update.

# The erase-install policy

The first piece is a Jamf Self Service policy that runs erase-install to kick off the upgrade. This is made available to all eligible computers so that an end user can upgrade on their own without needing to give them admin rights to do it through System Settings.

<figure>
  <img src="{{site.url}}/assets/images/erase-install-policy.png" alt="erase-install policy"/>
  <figcaption>The erase-install self service policy.</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/assets/images/policy-files-and-processes.png" alt="Calling erase-install in the policy"/>
  <figcaption>How to start erase-install from the policy after the package installs.</figcaption>
</figure>

Since this policy is for upgrades, the important parameters are `--os=14` and `--reinstall` to kick off an upgrade to MacOS 14. The other options are specific to how you might want the upgrade to behave. Check the [documentation](https://github.com/grahampugh/erase-install/wiki) to see what all the available flags do.

Enable the option to make this policy available in Self Service. This is all that is needed to allow users to start an upgrade on their own.

# Configuring Nudge to start the upgrade

There is an extra benefit to having the policy available: _The policy URLs._ Nudge has an [`actionButtonPath`](https://github.com/macadmins/nudge/wiki/userInterface#actionbuttonpath---type-string-default-value-nil) property that can be pointed to this URL.

<figure>
  <img src="{{site.url}}/assets/images/self-service-URI.png" alt="erase-install policy URI"/>
  <figcaption>Jamf Self Service URLs to directly open or execute the policy.</figcaption>
</figure>

Make a **_new_** Nudge configuration for this. Don't re-use one that is normally used for regular updates.

All the normal Nudge settings apply, especially the required minimum OS version and required installation date. The extra step is to have `actionButtonPath` pointed to the Installation URL for the policy. I should note that I did have this configured in both `userInterface` and`osVersionRequirements`, even though the documentation says that `osVersionRequirements` is the right place.

<figure>
  <img src="{{site.url}}/assets/images/nudge-sonoma-config.png" alt="config profile settings"/>
  <figcaption>This is where I set the Policy URL in the configuration profile.</figcaption>
</figure>

I chose to change the Nudge messaging slightly from what I normally use for minor updates, just to make sure there was an understanding that the process would take a little longer than an update usually does when the user sees the prompt. I also changed the `actionButtonText` to **Upgrade OS** instead of the default **Update Device**.

<figure>
  <img src="{{site.url}}/assets/images/nudge-major-upgrade.png" alt="Nudge upgrade window"/>
  <figcaption>This is what the Nudge window looks like with my configuration.</figcaption>
</figure>

Scope this Nudge config to any devices that haven't upgraded yet.

# Future Considerations

As of the time of me writing this, there is a major Nudge overhaul being worked by the developers. So big that they are bumping the version to 2.0. It is bringing the option to use [SOFA](https://sofa.macadmins.io) to automatically determine the latest versions of MacOS and let Nudge handle the deadlines using offsets and delays. All of the changes seem to be centered around SOFA support and all that comes with it. Nothing that I have seen in the changelog (and believe me, I have been paying _very close attention_ to this update) indicate any changes to make anything in this article work differently, but there is always a chance that there could be something in the future.
