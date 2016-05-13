The problem
===========

A full rsync of fedora-buffet0 can take hours just to receive the file list,
due to the fact that there are aroune eight million files.  And this has to
happen for every client, which is murderous on the download servers and worse
on the backend NFS server.

But the file list doesn't often change, and we know exactly when it does.  It
is trivial to pregenerate the file list on the server, but the rsync server has
no provision for making use of it.

We can't (or don't want to) change rsync, or write our own rsync server (which
might actually be a pretty cool idea for another day).

We can, however, generate a file on the server which the client can use to
determine exactly which files it wants.  This file needs to contain only a
timestamp (in seconds since epoch for easy parsing) and the filename.  The
timestamp should be the newer of the change time and the modification time.

The client can take this, figure out exactly what files it needs, and pass them
to rsync directly, skipping the lengthy "receiving file list" phase.

Client
======

* Store current time in a variable.

* Load the last mod time LASTTIME

* Download file/time list and hardlink list.

* Find all files and dirs with times newer than LASTTIME.

* Pass to rsync via ``--files-from``.

Server
======

After every operation which changes the filesystem:

* Recursively traverse the entire filesystem.

* Stat every file and directory.

* Find newer of mtime and ctime

* Write out time\tfilename
