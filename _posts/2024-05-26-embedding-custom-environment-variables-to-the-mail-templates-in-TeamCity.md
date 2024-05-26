---
layout: post
title: Embedding custom environment variables to the mail templates in TeamCity
description: Informative blog about embedding custom environment variables to the mail templates in TeamCity.
tags:
  - DevOps
---
Lately, I got the chance to setup fresh instances of TeamCity on Windows-based machines to let them act as build servers.

As you might have guessed, depending on the teams and games, specific requirements get added to the CI/CD pipeline. For one of the games, I needed to execute several AWS CLI commands to upload the build artifacts to the S3 bucket and then share that artifact with the team by using presigned URLs in TeamCity mail templates. Simply stored the presigned URL as an environment variable for the build and then embedded it in the mail notifications.

Figuring out how to edit custom environment variables during builds and embedding them in the mail templates was quite a journey that I enjoyed, and I will try my best to explain it with a mock scenario.

We will be creating a dummy environment variable for the TeamCity project, editing it during the build by using PowerShell, and then, at the final step, embedding it in the mail notification.

Let's get started!