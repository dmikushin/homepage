---
layout: post
title: "Working with Overleaf via Git"
tags:
- Overleaf
- Git
thumbnail_path: blog/2022-09-24-working-with-overleaf-via-git/overleaf.png
---

Overleaf deploys Git to track collaborative modifications to projects. Moreover, a user has an option to work with Overleaf's Git backend directly. It supports Git compatibility with some limitations: https-only, no force push. With some extra steps described below, it is possible to collaborate in development of an Overleaf paper for users working in the web-interface and in local git+pdflatex environments. The main advantages are of course a possibility to migrate an existing history of modifications to Overleaf, and an ability to manage a lot of files over the command line more efficiently.

In order to avoid repeated password typing, setup gpg password storage:

```
gpg --gen-key
pass init dmitry@kernelgen.org
```

Let's now clone an Overleaf repository with Git to see how it is organized:

```
export GCM_CREDENTIAL_STORE=gpg
git clone https://git.overleaf.com/632c23013b1a4f35618b0785
```

Note the new Overleaf repository will always have some initial commit, and force push is disabled. Therefore, in order to push some existing git repository to Overleaf, remove the existing dummy initial content and rebase your content on top of it. Consider a repository, which already contains your work in a master branch, then:

```
git remote remove origin
git remote add origin https://git.overleaf.com/632c23013b1a4f35618b0785
git branch -m your_existing_work
git fetch origin master
git checkout master
git rm -rf main.tex
git commit -m "Resetting Overleaf to an empty repository"
git checkout your_existing_work
git rebase -i master
git push origin master
git branch -D master
git branch -m master
```

The commands above basically rename your existing `master` branch to `your_existing_work`, then pulls a new `master` branch from Overleaf. Then we reset the state of the Overleaf's `master`, and rebase `your_existing_work` over it. Finally, the `master` branch is deleted, `your_existing_work` is renamed back to `master` and pushed to Overleaf.

