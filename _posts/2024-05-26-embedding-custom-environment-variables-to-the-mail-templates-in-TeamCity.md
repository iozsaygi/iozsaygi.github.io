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

### Adding a new environment variable
Well, at this point, I am pretty much assuming you have an up-and-running TeamCity server and agent instance(s) that you can use to try the things I explain.

First things first, we need to define our custom environment variable. You can see the environment variables by accessing the specific project that you defined. Within the project, you need to basically edit its build configuration. When you enter the build configuration page, you will see a menu that looks something like this:

![Build Configuration](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/ecevttmtit/build_configuration.png?raw=true)

Click on the parameters, and you will be greeted by a list of available parameters for your build. So far, the path looks like this: 
**Project | Project Configuration | Parameters**

- Inside the parameters window, click on the **'Add new parameter'** button.

![Add New Parameter](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/ecevttmtit/add_new_parameter.png?raw=true)

- It will open up a new popup window that requires you to fill in the in the properties of the new environment variable.

![Environment Variable](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/ecevttmtit/environment_variable.png?raw=true)

- To add a new environment variable, select the kind as **'Environment variable (env.)'**. I will be choosing **'Text'** as my value type and give it a name you prefer.

- After clicking on the 'Save' button, you should see the environment variable you've added in the parameters window. Mine looks like below after adding:

![Parameter Add Result](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/ecevttmtit/environment_variable.png?raw=true)


Well, we already accomplished most of the manual and effort-required steps with this work. Let's now take a look at the example PowerShell script that basically edits and saves our environment variable during build time.