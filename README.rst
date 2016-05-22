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
handle files which have been deleted on the server and files which are missing
on the client.  In most situations, it also allows hardlinks to be copied as
hardlinks instead of being downloaded multiple times.

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
new and updated files, deletes files and directories which no longer exist on
the server, and passes one combined list to rsync via --files-from.  Because
all modules are copied together, hardlinks between modules will be copied as
hardlinks.  Files and directories which no longer exist on the server are
deleted after the copy has completed, similar to the ``--delete-delay`` option
to ``rsync``.

The speed improvements can be extraordinary.  Just the "receiving file list"
phase of a mirror of ``fedora-buffet`` can take over ten hours and places a
huge load on the host from which you're downloading.  With this script it takes
six seconds.

Installation
------------

Copy ``quick-fedora-mirror`` somewhere.  Copy ``quick-fedora-mirror.conf.dist``
to ``quick-fedora-mirror.conf``, edit as appropriate and copy to one of the
following:

* /etc

* ~/.config

* The directory where quick-fedora-mirror lives

* The current directory when quick-fedora-mirror runs

* Anywhere you like, if you specify the path on the command line.

Options
-------

-a
    Always check the file list.  This disables the optimization in which the
    file list isn't processed at all if it hasn't changed from the local copy.
    Useful if you believe that some files have gone missing from your
    repository and you want to force them to be fetched.

-c
    Configuration file to use.

-d
    Set output verbosity.  See the VERBOSE setting in the sample configuration
    file for details.

-n
    Dry run.  Don't transfer or delete any content or update the timestamp.
    Note: the master is still contacted to download the file lists.

-N
    Partial dry run.  Ask rsync to do a normal transfer, but don't delete any
    local files which are not present in the file list.

-t
    Instead of the previous run time, use this many seconds since the epoch.
    Implies ``-a``.

-T
    Instead of the previous run time, use this.  The value is passed to ``date
    -d``, so it should be in a format which date recognizes.  ``yesterday`` and
    ``last week`` are useful examples.  Remember to quote if there are spaces.
    Implies ``-a``.

Initial Run
-----------

The last mirror time is assumed to be the epoch if ``quick-fedora-mirror`` has
not previously been run.  This means that every single file will be checked,
which will take many hours.  If you are certain your mirror is up to date, you
can just fudge the last mirror date::

    quick-fedora-mirror -T 'last week'

Then your run will only examine files which have changed in the last week.
This may still be a lot of files, but not all of them.

Adding a module
---------------

If you have to add a module after the fact (i.e. you already have
fedora-enchilada and you want to add fedora-alt), note that rsync will not pick
up any hardlinks.  You can of course do the download and then run hardlink
afterwards, or do a full transfer (i.e. using ``-t 0``).  To minimize the sime
spent counting files, you use a configuration file which specifies only the
modules with which the new module shares hardlinks.  Most of the hardlinks are
between fedora-archive and fedora-enchilada, or fedora-alt and
fedora-enchilada.

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

Downstream Mirrors
==================

Note that this method works for downstream mirrors as well.  Intermediate
mirrors should *not* modify the filelists.

Assuming ``rsync`` is called with --delay-updates, downstream mirrors should
always have a consistent view of the repository.  Changes should get out very
quickly, because mirrors can poll frequently without overloading servers.

Non-Fedora Usage
================
Note that you can of course run the server component in your own repository,
but the clients will of course need to specify ``REMOTE``, ``MASTERMODULE`` and
the ``MODULES`` array to map module names to directories.  The client also the
assumption that all of the separate module are included in a master module.  If
you would like to use this code but those constraints don't fit your use case,
please file an issue and I'll be happy to take a look.

Be sure to run ``create-filelist`` after every repository change.  You can also
run it from cron, but clients may see the repository in an inconsistent state
in the interval between the changes and the file list generation.  This will
not result in any repository corruption, though; clients will pick up the
correct repository state on the next run.

It's a good idea to run a diff or something and only copy the output into place
if the new output differs.  Technically ``create-filelist`` could just do this,
but it is currently easy to integrate into whatever script or arrangement you
might have in place.

Authorship and License
======================

All of this code was originally written by Jason Tibbitts <tibbs@math.uh.edu>
and has been donated to the public domain.  If you require a statement of
license, please consider this work to be licensed as "CC0 Universal", any
version you choose.
