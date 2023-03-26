---
layout: post
title: "Migrating from CVS to Git"
tags:
- Linux
- Git
thumbnail_path: blog/2023-03-26-migrating-from-cvs-to-git/TortoiseCVS_Logo.png
---

CVS is still around for many important projects, making it difficult to scale their development. Tutorials available for this topic are not robust enough for ease of use. Thus, here I would like to describe CVS to Git migration steps good enough for copy-pasting and just basic changes.

A very good starting point is given by [this post](https://svn.haxx.se/users/archive-2008-06/0631.shtml) for CVS repository of eCos:

```
rsync -r rsync://sourceware.org/ecos-cvs cvs > logs/rsync_out.txt 2> logs/rsync_err.txt
```

The point is we need to make a full copy (clone) of CVS repository, not just a checkout. Most likely, for CVS there always exists a copyable URL, but its path might be not so obvious, and knowing the right URL is the key.

The [cvs2git](https://github.com/mhagger/cvs2svn) tool still does a good job, but as of 2023 is already lost in the past. Therefore, I'm picking up a time-machine [cvs2git Docker container](git@github.com:jwiebalk/docker-cvs2git.git), which has Python 2 to run it:

```
docker pull jwiebalk/cvs2git
docker run -it -v $(pwd)/cvs:/cvs -v $(pwd)/git/repo:/git -v $(pwd)/log_output:/tmp/cvs_migration jwiebalk/cvs2git
```

Inside the Docker, we first navigate to `/cvs` folder to perform the dump, and then to `/git` to import the dump. By default, the final git state will be in "deleting all content", as when creating an orphanned branch. Therefore, we finish by doing `git reset --hard` and exit container:

```
cd /cvs
cvs2git --blobfile=/git/git-blob.dat --dumpfile=/git/git-dump.dat --fallback-encoding=ascii ecos >> /tmp/cvs_migration/ecos.log
cvs2git --blobfile=/git/git-blob.dat --dumpfile=/git/git-dump.dat --username=dmikushin --fallback-encoding=ascii ecos >> /tmp/cvs_migration/ecos.log
cd /git
git init ecos
cd ecos
cat /git/git-blob.dat /git/git-dump.dat | git fast-import
git gc --prune=now
git reset --hard
exit
```

On the host side, adjust the newly created Git repository ownership with `chown` and `chgrp`.

In principle that's all: Git tree is fully constructed and ready for pushing. But the log imported from CVS does not hold relevant information about the contributors:

```
git log | grep Author: | sort -u
Author: asl <>
Author: bartv <>
Author: grante <>
Author: gthomas <>
Author: jani <>
Author: jlarmour <>
Author: jld <>
Author: msalter <>
Author: nickg <>
Author: sergeig <>
Author: vae <>
```

So I'm going to use a Perl script to replace these names with better ones I've collected manually with a background search. The idea is to batch-process the Git repository with a `git filter-branch`, as shown in [this post](https://stackoverflow.com/a/870367/4063520):

```perl
#!/usr/bin/env perl -w

my(@authors) = (
    "asl",
    "bartv",
    "grante",
    "gthomas",
    "jani",
    "jlarmour",
    "jld",
    "msalter",
    "nickg",
    "sergeig",
    "vae"
);

my(@authors_new) = (
    # New names go here
);

my($i) = 0;
for ( ; $i < scalar(@authors); $i++)
{
    my($name) = $authors[$i];
    my($name_new) = $authors_new[$i];
    my($email_new) = $authors_new[$i];

    $name_new =~ s/\s*<.*$//g;
    $email_new =~ s/^.*<//g;
    $email_new =~ s/>//g;

    print "Renaming " . $authors[$i] . " to " . $authors_new[$i] . "\n";

    my($cmd) = "FILTER_BRANCH_SQUELCH_WARNING=1 " .
        "git filter-branch -f --env-filter '#!/bin/bash\n" .
        "OLD_NAME=\"$name\" \n" .
        "CORRECT_NAME=\"$name_new\" \n" .
        "CORRECT_EMAIL=\"$email_new\" \n" .
        "if [ \"\$GIT_COMMITTER_NAME\" = \"\$OLD_NAME\" ]; then \n" .
        "    export GIT_COMMITTER_NAME=\"\$CORRECT_NAME\" \n" .
        "    export GIT_COMMITTER_EMAIL=\"\$CORRECT_EMAIL\" \n" .
        "fi \n" .
        "if [ \"\$GIT_AUTHOR_NAME\" = \"\$OLD_NAME\" ]; then \n" .
        "    export GIT_AUTHOR_NAME=\"\$CORRECT_NAME\" \n" .
        "    export GIT_AUTHOR_EMAIL=\"\$CORRECT_EMAIL\" \n" .
        "fi \n" .
        "' --tag-name-filter cat -- --branches --tags";

    system("$cmd");
}
```

We simply execute this script to process the whole Git repo (may take several minutes):

```
perl ./authors.pl
```

Check out the nice new names with `git log`! Now the repository is really ready for pushing. Happy migrating!

