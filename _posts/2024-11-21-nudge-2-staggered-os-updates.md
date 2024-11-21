---
layout: single
classes: wide
title: "Staggered OS Updates with Nudge 2.0"
header:
  teaser: assets/images/default-image.jpeg
excerpt: "Nudge 2.0 can be used to stagger OS updates across a large fleet of macOS devices."
categories:
  - Mac Admin
  - Career
tags:
  - apple
  - nudge
  - mac
  - jamf
  - macos
permalink: nudge-2-staggered-os-updates
---

[Nudge](https://github.com/macadmins/nudge){:target="\_blank"} has been one of the most important and useful tools for me as a Mac admin. Nudge 2.0 brings powerful improvements that allow for better automation and control of macOS updates, and many of those changes inspired me to take our OS update strategy to a new level. When I was a Windows admin, our team established a staggered update process that had three waves (rings, cycles, phases, whatever you might call them) with a few -> many -> all approach. This allowed us to manage the impact of updates on our users and avoid overwhelming the help desk with issues that could be caught sooner. I wanted to do something similar for macOS, and Nudge 2.0 made it possible.

# SOFA

I want to first talk about [SOFA (Simple Organized Feed for Apple Software Updates)](https://sofa.macadmins.io){:target="\_blank"} and how it was the catalyst for this idea in the first place. SOFA is a website that monitors Apple's software update feed and provides information on when updates are available, what they contain, and whether they are critical or not. A JSON version of the data is also provided, which can be used to automate processes. What I wanted to do (and was about [95% of the way there](https://github.com/raydemay/nudge-remote-json){:target="\_blank"}) was create a script that would check SOFA for updates and then use Nudge to notify users when an update was available. This would allow me to control when updates were applied. Nudge supports remote JSON configurations, so I wanted to automate the update of this JSON based on changes to the SOFA feed. I would host the remote JSON somewhere and set Nudge to look at that. There would be different versions of the JSON for each wave (ring/cycle/phase), and each of these would have different deadlines. The one issue that I didn't fully answer was how to stagger the start of the waves. Then I saw what Nudge 2.0 was bringing and I knew it could solve my problem.

# OS Update Strategy

The update waves, which we ended up calling Patch Cycles, are as follows:

1. **Test**: A small group of devices that receive updates first, all IT staff that are either part of the endpoint team or are responsible for key services and applications.
2. **Pilot**: A "representative sample" of devices that receive updates next. These are both IT and non-IT staff, but with a focus on those who have critical workloads or use applications that might be impacted by the update.
3. **Production**: The rest of the devices.

<figure>
  <img src="{{site.url}}/assets/images/patchcycles.png" alt="Test>Pilot>Production Digram">
</figure>

These phases are used for both major and minor software updates, and in some cases we use them for phased rollouts of changes to the macOS configuration profiles that we apply to our devices or when a new feature is rolled out that requires additional testing. We use ServiceNow for our CMDB, and we have Patch Cycle as a custom attribute on each asset record. A script periodically writes this value to a Jamf extension attribute which is used to populate the smart groups used for scoping of the configuration profiles and policies.

# Nudge 2.0 and Minor Updates

There are a few key settings in Nudge that allowed me to establish our staggered OS update strategy. Most of these involve the SOFA feed and basing both the start of the notifications and the deadline off of when the OS update becomes available. [Here is a link to all of the updates.](https://github.com/macadmins/nudge/wiki/v2.0-features){:target="\_blank"}

These are some of the more important keys and explanations of what I set them to:

`optionalFeatures`

- `utilizeSOFAFeed` - This key sets Nudge up to check SOFA.
- `refreshSOFAFeedTime` - This is set to 12 hours instead of the default of 24 hours. The recommendation is to not make this too low as to not overload the service. I think the next phase of this would be for us to [host our own custom feed](https://sofa.macadmins.io/self-hosted.html){:target="\_blank"}.

`osVersionRequirements`  
I just have a single `osVersionRequirements` block that specifies the keys that set the requirements for each wave.

- `requiredMinimumOSVersion` - This is set to `latestMinor`, which means that Nudge will always require the latest minor version of the current macOS major version installed on that device.
- `standardMinorUpdateSLA` and `nonActivelyExploitedCVEsMinorUpdateSLA` - This setting is what controls the update deadline based on what CVEs are fixed in the update. I have this set to 1 day for Test, 3 days for Pilot, and 14 days for production.
- `activelyExploitedCVEsMinorUpdateSLA` - This is set to 10 days for Production. The test and pilot waves are fast enough that these are not set separately in those configurations.

`userExperience`

- `nudgeMinorUpdateEventLaunchDelay` - This setting delays the launch of Nudge. This is set to 7 days for Production and 1 day for Pilot. Combined with the SLA settings, this means that the notification for the Production wave starts 7 days after the update is released, then the deadline is 7 days after. Updates with fixes for actively exploited CVEs have a shorter deadline, so there will only be three days to update after the notifications begin.

The Nudge delay and the SLA/deadline calculations are independent. Nudge does not calculate the deadline off of the delay. It is solely based on the release date of the update. See [this GitHub issue](https://github.com/macadmins/nudge/issues/573){:target="\_blank"} for more details.

Here is a subset of the XML from my Jamf profile for the production deployment:

```xml
<key>optionalFeatures</key>
<dict>
  <key>acceptableUpdatePreparingUsage</key>
  <true/>
  <key>aggressiveUserExperience</key>
  <true/>
  <key>aggressiveUserFullScreenExperience</key>
  <true/>
  <key>asynchronousSoftwareUpdate</key>
  <true/>
  <key>attemptToCheckForSupportedDevice</key>
  <false/>
  <key>refreshSOFAFeedTime</key>
  <integer>43200</integer>
  <key>utilizeSOFAFeed</key>
  <true/>
</dict>
<key>osVersionRequirements</key>
<array>
  <dict>
    <key>aboutUpdateURL</key>
    <string>https://support.apple.com/100100</string>
    <key>activelyExploitedCVEsMinorUpdateSLA</key>
    <integer>10</integer>
    <key>minorVersionRecalculationThreshold</key>
    <integer>2</integer>
    <key>nonActivelyExploitedCVEsMinorUpdateSLA</key>
    <integer>14</integer>
    <key>requiredMinimumOSVersion</key>
    <string>latest-minor</string>
    <key>standardMinorUpdateSLA</key>
    <integer>14</integer>
    <key>targetedOSVersionsRule</key>
    <string>default</string>
  </dict>
</array>
<key>userExperience</key>
<dict>
  <key>loadLaunchAgent</key>
  <true/>
  <key>nudgeMinorUpdateEventLaunchDelay</key>
  <integer>7</integer>
</dict>
```

The production cycle Nudge profile is scoped to All Computers and then the pilot, test, and exception groups are excluded from this profile. There is a separate profile for the pilot and test groups.

In addition to the Nudge configurations, and to prevent devices from patching out of order, we do use software update deferrals that align with the Nudge delays. This does mean that I have three separate Restrictions profiles in Jamf for this (actually, I have 9 because of exceptions for other things), so that is something to think about if there is a desire to implement something similar. The deferral is an optional thing, but we wanted to be in tighter control of the waves in the event of an update getting released that causes a problem.

# Major Updates

I also use Nudge to do major updates, but this is a separate deployment. See [my other article]({{site.url}}/mac%20admin/nudge-eraseinstall/){:target="\_blank"} on how I do this. Since these only happen once a year, the SLA/deadline calculations are not as critical for major updates and a manual deadline is set. The major update process includes a communication plan, where we first announce that the upgrade is available in Self Service, then give information about the deadline. The deadline for the Production wave has been thirty days after the announcement for the last two major upgrades that I have done. Test is expected to update within one week of the release of the update, although some in the group may have already been on the beta. Pilot starts around 30 days after release, but may be sooner if early testing has gone well.
