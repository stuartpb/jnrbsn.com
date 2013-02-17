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

However, to clone SVN repositories, you need to use `git-svn`, which (as of git-svn version
1.7.12.2) doesn't support the `--mirror` option. However, you can get the same result by breaking
the action down to a few more component steps.

To start wih, initialize and enter a bare repository for your mirror:

    git init --bare sequel-pro.git
    cd sequel-pro.git
    
Once you've initialized the bare repository, run this command to prepare the repository for working
with the SVN repository.

    git --git-dir=. svn init http://sequel-pro.googlecode.com/svn
    
By default, `git svn init` is set up to keep all refs (branches and tags) from the SVN repository
separate from those that are local. This is not what you want for a mirror, so use these commands
to change the refspecs (paths to pull refs from and to) to something appropriate for a mirror:

    git config svn-remote.svn.fetch trunk:refs/heads/master
    git config svn-remote.svn.branches branches/*:refs/heads/*
    git config svn-remote.svn.tags tags/*:refs/tags/*

Once your Git repo is ready, you can fetch everything from the SVN repo with this command:

    git svn fetch

For most repositories, the above command will take a long time. It's basically iterating over every
single revision in every single branch of the subversion repository and committing them to the git
repository. The good news is that, once it's setup, subsequent syncs will _not_ take long at all
since it will only have to process the new revisions since last time. In the meantime, if you
haven't already done it, this would be a perfect time to login to your GitHub account and create a
new repository into which we can push all of this. Otherwise, go get a snack or something.

Once the repository has finished fetching everything, you can push it to GitHub by adding a
reference to the GitHub repository (a remote), and then pushing your main branch to the remote
repository on GitHub using the two commands below.

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
