Utilities for Mirroring Fedora Faster
=====================================

``quick-fedora-mirror`` comprises a suite of programs which together can be
used to propagate changes through a network of repository mirrors more quickly
than using rsync alone.

Background
----------

A full rsync of fedora-buffet0 (the main rsync repository which contains all
mirrorable Fedora content) can take hours just to receive the file list, due to
the fact that there are over 12 million files at the time .  This has to happen
for every client, every time they want to update, which is murderous on the
download servers and worse on the backend NFS server.  It also slows the
propagation of important updates because mirrors simply can't poll often.





Individual Documentation Pages
------------------------------

* `The Hardlinker <quick-fedora-hardlink.rst>`_
