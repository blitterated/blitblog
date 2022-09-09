---
date: 2022-08-02T03:59:00Z
title: "Replacing Committer Name and Email on Commits"
draft: true
image:
   hero: audio-vaporwave-fence.png
   thumb:
author: blitterated
tags: ["git", "git-filter-repo"]
categories: ["Software"]
---
Have you ever made a series of commits and forgot to change git's local config for your name and email?

Here we'll take a look at how to replace commits with the wrong name and email associated with them.

## Tools

### Install pipx with brew

```sh
brew install pipx
```

### Install git-filter-repo with pipx

```sh
pipx install git-filter-repo
```

## Operation

### Create a directory to work in and jump into it

This directory will be used to clone a repository into as well as to hold the `mailmap` file we'll use to remap the user and email address.

```sh
mkdir ~/fix-emails && cd $_
```

### Create a mailmap file in the work directory

If you try to do this in the repo itself, you get the following warning:

```Aborting: Refusing to destructively overwrite repo history since
this does not look like a fresh clone.
  (you have untracked changes)
Please operate on a fresh clone instead.  If you want to proceed
anyway, use --force.
```

The syntax is gitmailmerge.

```text
Proper Name <proper@email.xx> Commit Name <commit@email.xx>
```

Or another way to look at the syntax:

```text
Name-to-change-to <email@tochange.to> Name-currently-on-commits <email@tobechang.ed>
```

Now create the mailmerge file.

```sh
cat > mailmap <<EOF
blitterated <blitterated@protonmail.com> Real Name <realn@foo.bar>
EOF
```

### Pull down a fresh clone of your repo and cd into it

```sh
git clone git@github.com:blitterated/some-repo.git
cd some-repo
```

### Run git-filter-repo

```sh
git filter-repo --mailmap ../mailmap
```

### Check the results

```sh
git log
```

or `tig` if you've got it installed

```sh
tig
```

### Change the commiter name and email for this repo

```sh
git config user.name "blitterated"
git config user.email "blitterated@protonmail.com"
```

## Push

### Add the remote repo

```sh
git remote add origin ghblit:blitterated/some-repo.git
```

### Push to remote

```sh
git push -u --force origin master
```

### Pull and rebase in other local clones

```sh
git pull --rebase
```

## References

- [GH: git-filter-repo](https://github.com/newren/git-filter-repo#simple-example-with-comparisons)
- [git-filter-repo man page:User and email based filtering
](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html#_user_and_email_based_filtering)
- [gitmailmap(5) Manual Page](https://htmlpreview.github.io/?https://raw.githubusercontent.com/newren/git-filter-repo/docs/html/gitmailmap.html#_syntax)
- [How can I change the author name / email of a commit?](https://www.git-tower.com/learn/git/faq/change-author-name-email)
