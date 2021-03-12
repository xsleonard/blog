+++
title = "An alternative to hugo --cleanDestinationDir for git submodules"
tags = ["hugo", "git"]
date = "2020-01-07T00:00:00+00:00"
+++


This blog is created with [Hugo](https://gohugo.io) and published with Github Pages. The setup uses two repositories, [following the instructions from the Hugo docs](https://gohugo.io/hosting-and-deployment/hosting-on-github). 

One repo contains the Hugo content and themes files. The other contains the generated site. The former contains the latter as a submodule.

When a post is renamed or removed between site generations, unwanted files can be left behind in the generated site directory. To eliminate this problem, `hugo --cleanDestinationDir` will remove any extraneous files and folders from the destination folder, except for *folders* that begin with `.` (hidden folders). This behavior was added to avoid removing `.git/` folders. Unfortunately, git submodules contain a `.git` *file*, and will be removed when using `--cleanDestinationDir` (as well as any other hidden files, like `.gitignore`).

There is an [open (ignored) PR to fix this for the `.git` file](https://github.com/gohugoio/hugo/pull/6261). However, `.gitignore` would still be removed. To handle this, they'd need to add yet another exception for this file or expose yet another configuration parameter to allow the user to define which files to preserve. Both solutions are entering the "bloat" realm, so we're unlikely to see the change made.

As a workaround, all hidden files and folders can be ignored by modifying [the  `deploy.sh` bash script that is provided by Hugo in their documentation for Github Pages deployment](https://gohugo.io/hosting-and-deployment/hosting-on-github/#put-it-into-a-script).  

{{< gist xsleonard 2b08a9d71374f40b8ab6f1596b7fbfa1 >}}
