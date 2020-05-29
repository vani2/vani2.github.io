---
categories: xcode certificate ios
date: "2020-04-29T10:54:00Z"
title: Once again about code signing 

---

![Button angry person](/assets/2020-04-29/agutin.png)

> Signing your apps can be complicated. And we understand this.

Code signing becomes automatic since 2017 and Xcode 8 release. It’s 3 years ago. Before that was not the easy thing to manage certificate, provisioning profiles dances for many iOS engineers. I’ve eaten dog not once on it while working in a big team and someone can accidentally **revoke** your certificate. I blame Apple for this for years.

<!--more-->

These days this becomes much easier, but not everyone understands clearly how it works and what to do and not to do if you get sign error.

# Basics

![Certificate](/assets/2020-04-29/certificate.png)

What is a certificate? I mean by simple words. This is a file. This file contains two parts: public and private keys. By having this file you are identified as a privileged developer. If someone has the same certificate he can make the same privileged functions. That happens when you share your teammates the distribution certificate to give them the ability to upload builds into the AppStore. For security reasons, certificates only valid for a certain time period (from several months to several years).

![Certificate image](/assets/2020-04-29/keychain.png)

Private key underlined.

In Apple’s world we have a lot of certificates: development, distribution, push. Beginning Xcode 11 it’s available to generate Apple Development/Distribution certificate for all platforms instead of many certificates for every platform (iOS, macOS, tvOS).

# Development

Development certificate for the development. That’s it. If you enable automatic signing, Xcode will create a certificate for you. Just don’t forget to sign in to an account. Also, Xcode will add your device to development devices to developer.apple.com.

![Account Xcode](/assets/2020-04-29/apple-account.png)
![Debug settings](/assets/2020-04-29/debug-settings.png)

# Provisioning Profile

To distribute your app we need to create a provisioning profile. This also just file, that includes information:

* Application info: name, ID, platform, team name, creation, and expiration date.
* Entitlements.
* Info about a certificate(s).
* Devices (optional).

![Provisioning profile](/assets/2020-04-29/profile-1.png)
![Provisioning profile](/assets/2020-04-29/profile-2.png)

Devices info is obligatory for development and in house profiles. For App Store and Enterprise this section is empty. Just remember: **certificate is for a person** (team), **provisioning profile for an application**. The connection is one to many.

Also remember, if something changes (entitlements, a new device added), provisioning profile is invalidated and you need to regenerate it. To a quick look at a provisioning profile, I recommend installing [this tool](https://github.com/ealeksandrov/ProvisionQL).

# Distribution

Distribution is more sneaky. To distribute your app (to AppStore or Firebase whatever) you need manually create a certificate for it. And remember that the only two distribution certificates available for account. That is why you need to export a certificate for your teammate from the keychain. For sure, the better approach is to put a distribution certificate into CI Server. Also, you can use [match](https://docs.fastlane.tools/actions/match/#match), but IMO is less secure to put certificates to git (even private) repository.

To handle distribution signing I recommend to disable manual signing for Release. After that select created provisioning profile.

![Provisioning profile](/assets/2020-04-29/final-settings.png)

# Final recommendations

* Enable automatic signing for development and forget about certificates, profiles, and all this stuff.
* If you need to do a release build and get errors, check you have got a release certificate (with private key), provisioning profile, that created exactly with this certificate. Sometimes you have one of them and profile include info about another one.

For those who want to watch more, I highly recommend this [WWDC 2016 session](https://developer.apple.com/wwdc16/401).
