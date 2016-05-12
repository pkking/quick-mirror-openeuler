The problem
===========

A full rsync of fedora-buffet0 can take hours just to receive the file list,
due to the fact that there are aroune eight million files.  And this has to
happen for every client, which is murderous on the download servers and worse
on the backend NFS server.

But the file list doesn't often change, and we know exactly when it does.  It
is trivial to pregenerate the file list on the server, but the rsync server has
no provision for making use of it.

The obvious solution we can't use
=================================

If the content doesn't change, rsync could just cache the file list.  Or it
could somehow accept a pregenerated file list.  Escept that it doesn't do
that, and it's probably not within our purview to patch it to do so or write
our own server.  Although I have a feeling the latter isn't as bad as I think
it is.

Some upstream rsync talk:

https://lists.samba.org/archive/rsync/2009-March/022821.html
https://lists.samba.org/archive/rsync/2009-September/023909.html
https://bugzilla.samba.org/show_bug.cgi?id=3579

Solution constraints
====================

* We need to keep rsync on the server.

* We don't have the time or manpower to write our own rsync-compatible server
  or patch new functionality into rsync itself.

* We can run other arbitrary scripts on the server, though they must work in
  RHEL7 (so old python only).

* The client must be absolutely as simple as possibe.  Mirror admins don't want
  to install anything.  It really needs to reduce down to a bash script, though
  an initial implementation in python is perfectly reasonable.  I don't speal
  bash but a zsh script might be an interesting intermediate.

* It has to actually save some load on the master servers.

* We need to handle hard linking.  Fedora uses hardlinks all over the place.
  Hardlinks are difficult because:

  * Newly linked file(name)s have the same mtimes as the original ones, so the
    files appear to be new.  Thus a scan for new files won't find them.

  * If you naively transfer a directory with hardlinked files you may pull down
    a complete copy of things you already have.

  Any possible solition must avoid these; it should always transfer everything
  including newly hardlinked files, and must not make additional copies when
  the client already has the content.

Proposed solution
=================

Pregenerate a file list on the server.  This list needs to include file and
directory names, the modification time (seconds since epoch) and the inode on
the server.

Having directories enables the client to find new hardlinks by copying the
contents of directories newer than the last sync.  The containing directory's
mtime is updated even though the hardlinked files within appear to be old.

Having the inode lets the client determine files which may be hardlinked to
files it already has.  If rsync is told to transfer all of them, it should make
hardlinks as necesary without having to re-transfer things.  The server could
save the client some time by pregenerating tuples of hardlinks.



