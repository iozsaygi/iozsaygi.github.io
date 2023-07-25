---
layout: post
title: "Simple trick to clone large repositories"
description: "Story about chain of git commands to clone large repositories."
tag: DevOps and Build Engineering
---
Greetings everyone! 

This time I'll share a story about a small git operation that we automated with my colleague [İ. Çağkan Çağlayanel](https://crosline.github.io/) to setup large git repositories on build machines.

When I say large, I mean the 30–40 GB repositories that we work with. Well, it is probably nothing when you compare it to the AAA industry, but things are learned.

Here's the full set of git commands that we executed in order:
```bash
git config --global core.compression 0  
git clone --depth 1 <repo_url>  
git fetch --unshallow  
git pull --all
```

### Let's inspect each command one by one

``git config --global core.compression 0``

Setting ``core.compression`` to ``0`` will disable compression of objects in the git repository and objects will start to take up more space than before. Executing this command will disable compression for every repository on your system since this is editing the global git configuration. It will also remove the calculations for git object compressions.

``git clone --depth 1 <repo_URL>``

Cloning a repository with the ``--depth 1`` option will ensure that only the latest commit will be fetched, which saved us from unnecessary git commit history but it's good to keep in mind that it will not fetch the entire git commit history. So if you need to perform some operations in the git history, this will not help at all. (It's also called a shallow clone)

``git fetch --unshallow``

After cloning a repository with a limited commit history (shallow clone), we can fetch the entire repository's commit history by converting the repository to a complete clone. Sometimes we had to execute this command until our local repository matched the git revision.

``git pull -all``

After fetching complete history, we executed ``git pull -all`` to ensure everything was fetched and ready.

### Resources
[Here's](https://stackoverflow.com/questions/34389446/how-do-i-download-a-large-git-repository) the Stack Overflow thread that we explored during the investigation of this problem.

### Last words
Well, not that much but this trick saved us a lot of time. Automating this process by using batch scripts was even more fun. Especially when using source control manager in Jenkins build configuration.

I hope you enjoyed reading this as much as I enjoyed writing it. Thank you so much.