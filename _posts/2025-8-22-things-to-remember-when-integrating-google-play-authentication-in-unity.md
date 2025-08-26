---
layout: post
title: Things to remember when integrating Google Play authentication in Unity
description: Tricky things that can save time during Google Play authentication integration.
tags:
  - Unity
---
It's been quite a while since I updated this blog; lately I have been heavily working on authentication systems with [Google Play Games](https://docs.unity.com/ugs/manual/authentication/manual/platform-signin-google-play-games) and [Apple Game Center](https://docs.unity.com/ugs/manual/authentication/manual/platform-signin-apple-game-center).

I had the opportunity to conduct extensive research and tackle numerous tasks using [Google Cloud Console](https://console.cloud.google.com); I wanted to document my experiences here for future reference.

_Please keep in mind, this is not a full guide to implement Google Play Games authentication; instead, it is a list of important notes that might be useful._

To understand how authentication works on Google Play Games, first we need to take a look at the relationship between keystores and SHA-1 fingerprints.

## SHA-1 fingerprints and keystores
Whenever we build our Unity game for Android, our .apk is signed with a keystore. Whether it be the debug one that Unity automatically signed for you _(note that you can't upload this one to the Google Play Store)_ or the one you manually created.

Each time we use keystores, a unique SHA-1 fingerprint is also associated with the game build. Google actually uses this SHA-1 fingerprint to match their back-end services, such as authentication, leaderboards, achievements, etc., with your game.

_Considering you have [JDK](https://www.oracle.com/java/technologies/downloads/) available within your system, you can run the following command to learn more about your SHA-1 fingerprint within the keystore:_

`keytool -list -v -keystore your_keystore_name.keystore -alias your_alias_name`

Don't forget to save your SHA-1 fingerprint somewhere; you will need it heavily.

#### **Provide your SHA-1 fingerprint to OAuth credentials**
One of the essential steps is to provide your SHA-1 fingerprint for OAuth credentials within the Google Cloud Console; authentication will simply fail if you skip or forget this step, considering Google needs that SHA-1 fingerprint to match authentication with your game.

#### **Publish your Google Play Services project before testing authentication**
Within Google Play Developer Console, there is something called Play Services Project, which you need to complete setting up and publish (even if your game is not released on Google Play Store) for authentication to work. Otherwise, your mail account will be the only one that is able to authenticate in your game.

#### **You can update your SHA-1 fingerprint later on if you decide to change your keystore**
For some reason, if you update your keystore after the build is uploaded to Google Developer Console, you can just replace your OAuth credentials in Google Cloud Console with the new SHA-1 fingerprint that is available within your updated keystore.

#### **Leverage internal testers to validate authentication on multiple devices**
It is an amazing idea to have a group of internal testers within your organization and test the authentication by leveraging the internal test pipeline of Google Play.

_This case mostly works for the games that are not released yet; there are several additional things to watch out for when doing internal tests:_
* Authentication will fail if you are trying to authenticate on a device with a Google account that is not invited to test.
* Authentication will fail if the user hasn't created a gamer profile in Play Games Services yet; however, the native API will ask the user to create one even though players can skip this.

#### **Ensure you are signing with the correct keystore whenever authentication fails**
It is good to check if you used the correct keystore to sign your game whenever authentication fails. OAuth credentials require you to sign the game with the same SHA-1 fingerprint that you provided.

## Conclusion
Google Play Games authentication can get tricky if it is your first time tackling it; I built this list of small tips to help out for future references.

_I would like to share some resources that I used a lot:_
* [What is a SHA-1 fingerprint?](https://stackoverflow.com/questions/25685124/what-is-a-sha1-fingerprint)
* [Client authentication](https://developers.google.com/android/guides/client-auth)
* [Tutorial - Authentication with Google Play Games](https://discussions.unity.com/t/tutorial-authentication-with-google-play-games/911430)

See you!