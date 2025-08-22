---
layout: post
title: Things to remember when integrating Google Play authentication in Unity
description: Tricky things that can save time during Google Play authentication integration.
tags:
  - Unity
---
It's been quite a while since I updated here, trying my best to share new posts against my hectic work schedule. Lately, I have been working heavily on integrating authentication systems for Google Play (Android) and Game Center (iOS). This process taught me decent things, and I should not forget them in the future, so I decided to record them in a simple blog post. Also hoping that it will help me to get my hands back in this lovely blog again.

This post will be more text-oriented; you can think of it as some kind of checklist to remember when integrating Google Play authentication into your Unity game.

## Preface
If it's your first time integrating the authentication, it can be quite chaotic for Google Play because there are many things moving at the same time. Fear not, I would like to share some tricky details that might help you during this. 

_Keep in mind that this is not a full guide; it is just notes for tricky steps that you might miss during integration._

## What exactly are the SHA-1 keys?
SHA-1 is basically an algorithm that helps us encrypt our data. I don't know how this algorithm exactly works yet, but whenever we sign our Unity game with a keystore, whether it be the debug one or the release one that you created manually, an SHA-1 key is also generated that is related and specific to your keystore.

_You can run the following command (requires JDK) to learn the SHA-1 fingerprint of your keystore:_
`keytool -list -v -keystore your_keystore_name.keystore -alias your_alias_name`

After executing this command, you will be asked to enter the keystore password and alias password within the terminal.

SHA-1 keys become pretty important when we are adding OAuth credentials to our Google Cloud project, which is a required step to achieve Google Play authentication.

## Tricky checklist
1. `You will have to enter your SHA-1 fingerprint into the OAuth credentials.`
	* We can't skip this step considering Google Play needs to match their authentication services with your game. This is where SHA-1 fingerprints come in; you will need to provide the SHA-1 fingerprint from your keystore to the OAuth client.
</br>
2. `Don't be like me and publish your Google Play Services project before testing authentication.`
	* Okay, this sounds silly, but I think I lost around 4-5 hours trying to figure out why my authentication does not work. At the end of the day, I realized that I forgot to press the publish button in Google Play Console. Yep, it happens.
</br>
3. `You don't need to recreate your Google Play Console project if you decide to change your keystore, which also changes your SHA-1 fingerprint.`
	 * I created a dummy keystore, and I eventually forgot the password and alias of it. This led me to change the keystore of the game, which also changed the SHA-1 fingerprint. This immediately broke the authentication because my OAuth credentials were relying on the old SHA-1 fingerprint. I totally forgot about that and was considering recreating the Google Play Console project; luckily, I realized that updating the OAuth credentials with the new keystore's SHA-1 fingerprint is enough to work this around.
</br>
4. `You can create a list of internal testers and test authentication with them by using Google Play's app distribution pipeline.`
	* It is always good to test authentication on multiple devices, and internal test releases are a cool way to achieve this. Just include the people's e-mail that you want, and they will be able to download your game and check if authentication works even if the game is not released on the Google Play Store yet.
</br>
5. `Don't forget to embed OAuth resources from Google Play Console to the Unity editor by using the Google Play Games plugin.`
	* After you feed your SHA-1 fingerprint to the Google Play Console, it will generate resource data for you to embed into Unity by using the Google Play Games plugin. It is a good idea to check this embedded data in the Unity editor from time to time, considering it gets reset by some kind of editor bug after producing Android builds.
</br>
6. `Even though you set up everything correctly, authentication will still fail if the following conditions are not met in the test device:`
	* You have to bind a Google account to your device for authentication to succeed.
	* If there is no gamer profile created in the Play Games application, the authentication will ask you to create one. It will eventually fail if you decide to not create a gamer profile.
	* If you are testing via internal test builds (the case where the game is not publicly released yet), authentication will fail if somehow you are trying to log in with a Google account that is not invited to test.
	* If you build your game with a different keystore with a different SHA-1 fingerprint than the one you have in OAuth credentials, the authentication will fail because Google will not be able to match your game, considering the SHA-1 fingerprint has changed.

## Conclusion
It took me around two or three days to fully integrate Google Play authentication. It would definitely have taken shorter if I had known these tips beforehand. Sharing these tips, hoping it will help someone out there, or at least will help me in the future if I try to integrate this again at some point.

Also, I am strongly hoping that this will be a good entry to get my hands back into blogging, but it still depends on my schedule.

Thank you for reading this out!