---
layout: post
title: Detaching from Expo
date: 2019-01-14 15:13 -0500
---

I've been using Expo for most of my iOS and Android development as it offers the benefits of React Native while abstracting even more of the permissions, OTA update ability, and build processes. However, on a recent project, I needed access to a lower level library to allow for PDF rendering (and control, etc.) inside of the app, so I had to bite the bullet and perform `expo-cli eject`. Overall, the process has been fairly straightforward. One issue I did run into was needing to get a copy of my iOS certificates so I could upload the binary I built to the iOS store. After some fiddling, I was able to get access to it by doing the following:

```
expo-cli fetch:ios:certs
```

This gave an output like:

```
[15:07:44] Retreiving iOS credentials for @jacobwesleysmith/digapub
[15:07:45] These credentials are associated with Apple Team ID: <Redacted>
[15:07:45] Writing distribution cert to /Users/jacobsmith/github/jacobsmith/magazine-mobile/digapub_dist.p12...
[15:07:45] Done writing distribution cert credentials to disk.
[15:07:45] Writing push cert to /Users/jacobsmith/github/jacobsmith/magazine-mobile/digapub_push.p12...
[15:07:45] Done writing push cert credentials to disk.
[15:07:45] Writing provisioning profile to /Users/jacobsmith/github/jacobsmith/magazine-mobile/digapub.mobileprovision...
[15:07:45] Done writing provisioning profile to disk
[15:07:45] Save these important values as well:

Distribution p12 password: <Redacted>
Push p12 password:         <Redacted>

[15:07:45] All done!
```

At this point, the profiles, keys, etc. were downloaded on to my local file system. The part that I missed at first was the `password` values that it printed out. Dragging and dropping the keys into the app 'Keychain Access' prompted me for a password, which is the value that it printed out after downloading the certs.

Once the password was entered, the keys were added to my local store, Xcode was able to read them, and signing the app and uploading went off without a hitch!
