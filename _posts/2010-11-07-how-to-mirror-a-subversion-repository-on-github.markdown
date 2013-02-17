---
layout: post
title: How to mirror a subversion repository on GitHub
description: How to setup and maintain a mirror of a subversion repository on GitHub.
categories:
  - automation
  - git
  - how-tos
  - tips
keywords:
  - automation
  - cron
  - git
  - git-svn
  - github
  - how to
  - mirror
  - repository
  - script
  - subversion
  - svn
---
If you use [GitHub](http://github.com/), you may know that they provide an easy way to import an
existing [subversion](http://subversion.apache.org/) repository. However, they don't provide a way
to _mirror_ a subversion repository on GitHub and keep it in sync with the original. When I did this
for the first time, it took me awhile to figure it out, but once I did, I was surprised how easy it
was. So now I'm going to share it with you.

<!--more-->

Before you do this, you'll need an intermediary computer on which you can do the automated syncing.
This guide assumes that computer is running Linux/Unix. Basically, any always-on *NIX machine will
do the job. This machine is where all of the commands below should be run. In this example, I will
be creating a mirror of the source code of [Sequel Pro](http://www.sequelpro.com/), a very nice
MySQL database management app for Mac OS X. To do this with a different repository, just change the
paths/URLs where appropriate.

First, you need to clone the repository to your machine. Normally, when creating a mirror of a Git
repository, you would use `git clone --mirror`, which sets the repository up to only keep the Git
data (a "bare" repository), maps all refs (like branches and tags) locally instead of keeping them
as remote references (ie. prefixing them with "origin/"), and configures the repository to map all
refs locally in the future.

However, to clone SVN repositories, you need to use `git svn clone`, which (as of git-svn version
1.7.12.2) doesn't support the `--mirror` option. You can work around this by converting the
repository to be like a repository created with `git clone --mirror` after the fact. First, run the
command below to create a normal Git clone of the SVN repository.

    git svn clone --stdlayout http://sequel-pro.googlecode.com/svn sequel-pro

The `--stdlayout` command tells `git svn clone` to fetch the master branch from `trunk`, the other
branches from `branches`, and the tags from `tags`. If the SVN repository is using another,
non-standard layout, you can specify the paths with `-T`, `-t`, and `-b` arguments, respectively
(see the [git-svn man page][]).

For most repositories, the above command will take a long time. It's basically iterating over every
single revision in the subversion repository and committing them to the git repository. The good
news is that, once it's setup, subsequent syncs will _not_ take long at all since it will only have
to process the new revisions since last time. In the meantime, if you haven't already done it, this
would be a perfect time to login to your GitHub account and create a new repository into which we
can push all of this. Otherwise, go get a snack or something.

Once it's done setting up the repository locally, the first step to make the repository into a
mirror is to make it bare. A bare repository is just the `.git` directory from a regular Git
repository, with an additional config option to stop Git from trying to find and use a working copy
for development. To start, move the .git directory out of the working directory to stand on its
own: conventionally, the bare equivalent of a non-bare Git repository is the name of the repository
followed by `.git` (you can see this at the end of GitHub URLs, for instance).

    mv sequel-pro/.git sequel-pro.git
    
At this point you may remove the working copy of the files made during `git svn clone`:

    rm -rf sequel-pro # NOT sequel-pro.git! That would delete the repository!

After you've moved the Git repository, you need to change to the repository directory and tell Git
that it's a bare repo now:

    cd sequel-pro.git
    git config core.bare true
    
Now you need to change the refs made by `git svn clone` to map to the local repository. There are
two parts to this: first, you must move the existing refs, then you modify the refspecs in the
config section for the SVN remote.

    mv refs/remotes/tags/* refs/tags
    rm -r refs/remotes/tags/ # You don't want to accidentally move your tags into branches
    rm refs/remotes/trunk # remotes/trunk will be obsolete to master after making these changes
    mv refs/remotes/* refs/heads/
    git config svn-remote.svn.fetch trunk:refs/heads/master
    git config svn-remote.svn.branches branches/*:refs/heads/*
    git config svn-remote.svn.tags tags/*:refs/tags/*

Once you've finished converting the repo to a mirror, you can push it to GitHub by changing to
the directory to which you cloned the repository, adding a reference to the GitHub repository
(a remote), and then pushing your main branch to the remote repository on GitHub using the three
commands below.

    git remote add --mirror=push origin git@github.com:jnrbsn/sequel-pro.git
    git push origin

Now that you have the mirror setup, you need to setup a way to have it automatically synced with the
original subversion repository. If you're using Linux/Unix, you can do this with a simple shell
script setup up as a cron job. The script would look something like the one below. The script will
first change to the appropriate directory, then use the `git svn fetch` command to pull any changes
from the subversion repository, and finally, push the changes to GitHub.

    #!/bin/bash

    cd /path/to/sequel-pro.git/
    git svn fetch
    git origin

The easiest way to set this up as a cron job is to simply put the script in one of the
`/etc/cron.daily`, `/etc/cron.hourly`, etc. directories (make sure it's executable). Or you could
create an entry for it in your crontab (which I won't get into since it could take up it's own whole
post. Google it— I'm sure you'll figure it out).

That's it. Happy mirroring!

[git-svn man page]: http://www.kernel.org/pub/software/scm/git/docs/git-svn.html
