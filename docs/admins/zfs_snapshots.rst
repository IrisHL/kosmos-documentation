===================
Snapshot management
===================

Rhea ZFS filesystems
====================

Rhea is one of the storage nodes of the Kosmos cluster. Its storage is organized using ZFS: the physical disks are combined into a shared storage *pool*, and ZFS filesystems (a type of *dataset*) allocate space dynamically from that pool. Their size does not need to be fixed in advance, although quotas can limit how much space they use and reservations can guarantee that a certain amount remains available to them.

All project folders and home folders on Rhea live in the ZFS pool named ``project-pool``:

- ``project-pool`` (ZFS root filesystem)

  - ``network_homes`` (ZFS child filesystem): stores all the home folders + their snapshots (children of ``network_homes``)
  - ``projects`` (ZFS child filesystem): stores all the project folders (children of ``projects``
  - ...

Existing filesystems can be listed with:

.. code-block:: bash

    zfs list <path to folder>       # a single filesystem
    zfs list -r <path to folder>    # recursively

Home folders
============

Creating a new home folder
--------------------------

.. code-block:: bash

    sudo create-home -u <username>

This runs ``/usr/bin/create-home``, which creates the user's home filesystem and sets its storage limits, so these do not need to be set manually.

Home folder quotas and snapshots
--------------------------------
The live files of the user may occupy up to 150 GiB (quota), while the live filesystem and its snapshots may together occupy up to 300 GiB (refquota). These snapshots can help recover files that a user accidentally changes or deletes.

- ``refquota=150G``: the user's live files may occupy up to 150 GiB.
- ``quota=300G``: the live filesystem **and** its snapshots together may occupy up to 300 GiB.

Snapshots help recover files that a user accidentally changed or deleted.

.. warning::

    A snapshot is **not** a true backup: it lives on the same storage pool as the original data. If the pool or its underlying hardware fails, both the live files and the snapshots are lost. Important data should also be covered by the tape backups.

.. note::

    The 150 GiB for live files is not reserved exclusively for them. As snapshots grow, the space available for new files shrinks, so to the user it looks like their home folder is getting smaller. The 150 GiB limit has not changed, but part of that capacity is unavailable because the shared 300 GiB limit is being reached. Deleting old snapshots (see `Removing snapshots`_) may restore the available space.

What does a snapshot store?
---------------------------

ZFS snapshots do not contain full copies of files. They are read-only views of a filesystem at a particular moment, built from block pointers: when a snapshot is created, ZFS records references to the blocks holding the filesystem's data and metadata.

ZFS uses copy-on-write. When data in the live filesystem changes, ZFS writes the new version to different blocks instead of overwriting the originals. The snapshot keeps referencing the old blocks, which preserves the older version of the data. This is why snapshots consume more space as files in the live filesystem are changed or deleted. Blocks may be shared between the live filesystem, snapshots and clones.

.. note::

    Deleting a snapshot does not necessarily free all the space visible through it. ZFS only frees blocks that are no longer referenced by the live filesystem, another snapshot or a clone. Deleting one snapshot may therefore free little space if its blocks are still shared with other snapshots; a block only becomes free once its last reference is removed.

How snapshots are organized
---------------------------

Snapshots of a home folder can be listed with:

.. code-block:: bash

    zfs list -t snapshot project-pool/network_homes/<username>

Snapshots are taken on a daily, weekly and monthly basis, indicated by the name prefixes ``daily-``, ``weekly-`` and ``monthly-`` followed by the date, e.g. ``project-pool/network_homes/<username>@weekly-2026-06-15``.

They are created automatically by root's crontab on Rhea:

.. code-block:: text

    # m h  dom mon dow   command
    0 2 * * * /root/daily_zfs_snapshots.sh
    0 2 * * 1 /root/weekly_zfs_snapshots.sh
    0 2 1 * * /root/monthly_zfs_snapshots.sh

- ``/root/daily_zfs_snapshots.sh``: every day at 02:00
- ``/root/weekly_zfs_snapshots.sh``: every Monday at 02:00
- ``/root/monthly_zfs_snapshots.sh``: on the first day of each month at 02:00

Each script recursively snapshots all home folders. For example, the daily script:

.. code-block:: bash

    #!/bin/sh

    # Get the current date as a snapshot label
    date_label=$(date "+%Y-%m-%d")

    zfs snapshot -r "project-pool/network_homes@daily-${date_label}"

How users access their snapshots
--------------------------------

Users can browse their snapshots in the ``.zfs/snapshot`` directory inside their home folder on Kosmos (e.g. ``ls ~/.zfs/snapshot``). The ``.zfs`` directory is a hidden virtual directory provided by ZFS, and snapshots are mounted automatically when accessed. Snapshots are read-only, so users cannot modify or delete them. To recover a file, users copy the earlier version from ``.zfs/snapshot/<snapshot name>/`` back into their home folder.

Managing snapshot storage space
-------------------------------

To see how much space each home folder's snapshots use:

.. code-block:: bash

    zfs list -o space -r project-pool/network_homes

The ``USEDSNAP`` column shows the space used by snapshots, ``USEDDS`` the space used by the live filesystem, and ``AVAIL`` the space still available to it.

Removing snapshots
------------------

Semi-automated pruning
~~~~~~~~~~~~~~~~~~~~~~

Clone `zfs-prune-snapshots <https://github.com/bahamas10/zfs-prune-snapshots>`_ (v1.5.0) and run, for example:

.. code-block:: bash

    sudo ./zfs-prune-snapshots -v -p daily 8d project-pool/network_homes
    sudo ./zfs-prune-snapshots -v -p weekly 1M project-pool/network_homes
    sudo ./zfs-prune-snapshots -v -p monthly 3M project-pool/network_homes

This removes all daily, weekly and monthly snapshots older than 8 days, 1 month and 3 months respectively. See the README on GitHub for all options.

Running this periodically (about once a month) is usually sufficient. It could also be automated.

Removing individual snapshots
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Despite regular pruning, snapshots can grow quickly if many large files are changed or deleted. In that case, additional snapshots may need to be removed manually:

.. code-block:: bash

    sudo zfs destroy project-pool/network_homes/<username>@<snapshot name>

for example ``sudo zfs destroy project-pool/network_homes/<username>@daily-2026-09-11``.

To preview the operation without deleting anything, first do a dry run (``-n``) with verbose output (``-v``):

.. code-block:: bash

    sudo zfs destroy <path to snapshot>

Example:

.. code-block:: bash

    sudo zfs destroy -nv project-pool/network_homes/<username>@daily-2026-09-11

.. danger::

    Always include the ``@<snapshot name>`` part. Be careful not to destroy the home folder itself, e.g. do **not** run ``sudo zfs destroy project-pool/network_homes/<username>``. To preview the operation without deleting anything, use -n for a dry run and -v for verbose output.

Project folders
===============

Project folders do **not** have snapshots.
