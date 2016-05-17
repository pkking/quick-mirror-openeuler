Utilities for mirroring Fedora faster
=====================================

A full rsync of fedora-buffet0 can take hours just to receive the file list,
due to the fact that there are over 12.4 million files.  This has to happen for
every client, every time they want to update, which is murderous on the
download servers and worse on the backend NFS server.  It also slows the
propagation of important updates because mirrors simply can't poll often.

By generating a simple database of file and directory timestamps, it becomes
easy for the client to determine which files have changed since the last mirror
run and only mirror those files.  This also provides enough information to
handle files which have been deleted on the server.  And if the timestamps are
chosen properly, it allows hardlinks to be copied as hardlinks instead of
copying the files multiple times.

Client
======

The client is ``quick-fedora-mirror``.  It is currently written in zsh but
should be portable to bash.  (I needed associative arrays and haven't learned
to do them in bash.)  It needs no external tools besides ``rsync``, ``awk``,
and the usual core utilities.

A config file is required (unless you edit the script); see the sample file in
``quick-fedora-mirror.conf.dist``.  The destination directory and location of
the file to store the last mirror time must be set, though you probably want to
set the list of modules to mirror as well.

The client downloads the master file list for each module, generates lists of
new files, deletes files and directories which no longer exist on the server,
and passes one combined list to rsync via --files-from.  Because all modules
are copied together, hardlinks between modules will be copied as hardlinks.

The speed improvements can be extraordinary.

Server
======

The server must include one file (by default, fullfiletimelist) per module to
be mirrored using this code.  This file is created by running
``create-filelist``.  This will generate a list of all files in the current
directory in the proper format and write it to ``STDOUT``.  This output should be
captured to a tempfile and moved into place once the run is complete.  It could
do that directory but I wanted to keep it simple.

The timestamp in the file list is the newer of mtime and ctime.  This means
that newly created hardlinks will cause both the original and the new version
of the file to appear to have been updated.  ``rsync`` will note that the extra
files are up to date and will create the hardlinks directory (assuming, of
course, that it is called with ``-H``).

The format of the file list is simple enough to be parsed by a shell script
with a few calls to awk.

Be sure to run ``create-filelist`` after every repository change.

Downstream Mirrors
==================

Note that this method works for downstream mirrors as well.  Intermediate
mirrors should *not* modify the filelists.  Even though the timestamps
(specifically, ctime) may differ, the file list is still a valid source of data
even for mirrors far down the chain.  Assuming ``rsync`` is called with
--delay-updates, downstream should always have a consistent view of the
repository.  Changes should get out very quickly, because mirrors can poll
frequently without overloading servers.
