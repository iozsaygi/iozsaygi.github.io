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

- After clicking on the **'Save'** button, you should see the environment variable you've added in the parameters window. Mine looks like below after adding:

![Parameter Add Result](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/ecevttmtit/parameter_add_result.png?raw=true)

Well, we already accomplished most of the manual and effort-required steps with this work. Let's now take a look at the example PowerShell script that basically edits and saves our environment variable during build time.

### Adding a new PowerShell script to set custom environment variables during the build
We need to add a new build step to work with PowerShell scripts. Let's go to **Project | Build Configuration | Build Steps** and then press the **'Add build step'** button.

Choose **PowerShell** as runner type and give your build step a proper name. Also, don't forget to change **'Script'** to **'Source code'** so you can directly edit the script from the TeamCity interface. The other option will let you use the scripts that are located on the machine.

Here is our little script, pretty straight forward and on purpose:
```shell
Write-Host "##teamcity[setParameter name='env.Person' value='İsmail Özsaygı']"
Write-Output $env:Person
```

Let's explain it line by line:
1. The first line actually finds the environment variable that we defined in the build configuration and updates its value as 'İsmail Özsaygı' which is my name and surname.
2. The second line is just for debugging; it prints out the environment variable that we just sent to let us debug if it is working correctly.

One thing to note is that once the build ends and a new build starts, the environment variable we modified will be reset to its default value.

Now, let's see how we can edit our mail templates to show that environment variable on our mail notifications. But before that, I'll assume you've already setup a mail server and notifications by using the TeamCity interface.

### Editing the mail template to show a custom environment variable
Now we need to edit the mail templates that are defined in TeamCity. You can access the templates from **Administration | Global Settings** and click on the **'Browse'** button near the **'Data directory'** section. It will list the data directory for TeamCity in the interface. Then go to **config | notifications | email | build_successful.ftl**, and it will open a simple text editor in the TeamCity interface for you to edit the mail template.

One thing to note is that we are only going to edit the notifications that are triggered upon successful builds; it is up to you and your workflow to edit different types of templates.

I only added the following line of code to the ``<div>`` tag under ``<#global bodyHtml>``, and it worked like a charm!

``<b>Person:</b> ${build.parametersProvider.all['env.Person]}``

And here is our new mail that I just received!
![Mail result](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/ecevttmtit/mail_result.png?raw=true)

Well, that's it. We learned how to add a small but really effective feature to our TeamCity CI pipeline. What you can do with this is pretty much up to your imagination and project requirements.
### Conclusion
Now that was pretty niche content, I assume! It was really fun to tackle with TeamCity, and I definitely had a smoother experience than Jenkins. Maybe I'll write a blog post about the differences between TeamCity and Jenkins one day.

Thank you for reading it through!
