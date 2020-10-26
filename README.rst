smart-live-rebuild
==================


smart-live-rebuild is a Python script to aggregate live packages
from users' system, update them and merge the updated ones.


Motivation
----------
What's the point in creating such tool when portage-2.2 comes
with builtin ``@live-rebuild`` support?

The @live-rebuild set of portage and similar tools simply do the task
of aggregating the live packages and calling emerge to update them. This
is acceptable if you don't have many live packages and don't call it
frequently.

But if you used a whole bunch of live packages (like live X11), you'd
probably get a little irritated having to rebuild all of them, even
if most of them weren't even touched by upstream.

Of course, there's ``LIVE_FAIL_blah_blah`` in git.eclass. It works indeed
but have you used it with, say, more than 90 packages? You can imagine
how long it takes for portage to reload itself 90 times after dying
on each non-modified repo.

And that's where smart-live-rebuild comes in handy. It updates all
the live packages on its own, and supplies portage with ready-made list
of packages which indeed have changed and need to be rebuilt.


Features
--------
1. Parallel updates support

   As you might have already noticed, fetching from remote repositories
   suffers from a constant start-lag, wasting a lot of time without
   really consuming network bandwidth.

   To overcome that, smart-live-rebuild support running multiple
   updates in parallel. Although it might seem insane at first,
   assuming that most repositories aren't really changed or have quite
   a small set of changes, using it could save a lot of time.

   To use parallel updates, pass the `-j N` (`--jobs N`) parameter
   to s-l-r, where N stands for the number of updates supposed to be
   running in parallel.

2. Quickpkg backup support

   If you used a large set of live packages for some time, then you
   probably met an issue where the new upstream version was broken,
   and you had to make an effort in order to find a working commit
   to revert to.

   This is where the ``-Q`` (``--quickpkg``) option becomes useful.
   It makes s-l-r call ``quickpkg`` to create the binary packages
   for the current versions of packages queued to be updated.

   Although it wastes some time in each update, it allows you to easily
   and quickly revert to the previous working version of the package --
   without unnecessarily recompiling it. Moreover, the binary packages
   contain a copy of environment.bz2, allowing you to get the last
   working commit/revision.

3. Real update checking

   s-l-r doesn't assume it is the only application which can update
   the repository. Whenever possible, it tries to retrieve
   the currently-installed commit (revision) from vardb
   instead of relying on the repository state before update.

   This way, the rebuild list will contain all packages which
   are outdated in the system and not only those which it has updated
   in the particular run.  This means you no longer have to worry
   about having to manually update packages whenever you abort
   a merge or update process.


Security
--------
In most cases, s-l-r needs to be run as root. This has serious security
implications, which the scripts tries to overcome dropping
the privileges whenever possible.

In fact, the superuser privileges are only required to call emerge
and quickpkg (the latter might in some cases work as regular user
but that has further implications). It is also required to update
the repositories unless userpriv is enabled (otherwise, portage
privileges are enough).

s-l-r loses its privileges as soon as configuration and command-line
options are parsed (this is required in order to support disabling
the privilege dropping), and performs the repository updates
as the ``portage`` user (as long as userpriv is enabled).

If ``quickpkg`` is scheduled and/or ``--pretend`` is not being used,
s-l-r forks to drop the privileges and performs the updates using forked
subprocess. Otherwise, it directly drops the privileges in the parent
process.

Moreover, in the latter case s-l-r can be run directly by the ``portage``
user.


Portage set support
-------------------
Apart from being called directly, smart-live-rebuild provides a package
set for portage-2.2, called ``smartliverebuild.sets.SmartLiveRebuildSet``.
Please take a look at ``sets.conf.example`` file for a use example.


Filtering
---------
The list of packages to be updated can be filtered using
``--filter-packages`` (``-f``) option. This options takes a *single*
argument being a wildcard either to the package name alone,
or category/pn. Multiple filters can be passed as additional
``-f`` options or ``,`` separated. A filter can be prefixed with ``!``
to make it exclusive. Otherwise, the filter is inclusive.

The filters are applied left-to-right. Exclusive filters remove packages from
currently matched set, inclusive (re-)add them. If the first filter
is inclusive, the set is initially empty; otherwise, it contains all packages
by default.

Example filter sets:

1. ``-f app-portage/*`` -- update only packages in ``app-portage`` category,
2. ``-f !python`` -- update everything except for ``python``,
3. ``-f app-portage/* -f !app-portage/smart-live-rebuild`` -- update packages in
   ``app-portage`` category, except for ``smart-live-rebuild``,
4. ``-f f* -f !app-portage/* -f app-portage/flaggie`` -- update all packages whose
   names start with ``f``, except for those in ``app-portage`` category but still
   update ``app-portage/flaggie``.


Configuration file
------------------
Various options to smart-live-rebuild may be set in a configuration file
too. The path to that file can be specified as ``--config-file``
(or ``-c``), and the default one
is ``/etc/portage/smart-live-rebuild.conf``.

The configuration file has a format similar to Windows .ini files.
Sections correspond to available profiles, with ``[smart-live-rebuild]``
being the default one (other can be specified using ``--profile``/``-p``).
The keys for respective options match the command-line argument names,
with ``no-`` prefix stripped and all dashes (``-``) replaced with
underscores (``_``).

The values for boolean types can be specified in any way suitable
for ConfigParser python class (i.e. true/false/yes/no/on/off).
The ``type`` list has to be comma-separated.

An example config file may look like::

	[smart-live-rebuild]
	jobs=3
	erraneous_merge=no
	type=git,subversion

Additionally, configuration files can be chained. To do so, just specify
path to the next configuration file as ``config_file``. Please notice
though that currently they aren't read in reverse order, so further
configuration files will replace values set by yours.


Bug reporting
-------------
Please report bugs either to `the issue tracker`_ or `Gentoo Bugzilla`_.

.. _the issue tracker: https://bitbucket.org/mgorny/smart-live-rebuild/issues/
.. _Gentoo Bugzilla: https://bugs.gentoo.org/

.. vim:syn=rst
