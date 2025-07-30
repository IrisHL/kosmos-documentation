=======================================================
Creating and managing storage (home, project, and data)
=======================================================

This guide provides instructions to **create, manage, and control access** to storage directories across ``/home``, ``/projects``, and ``/data`` using the provided admin scripts.

Each script handles both **directory structure setup** and **access permissions using ACLs**, ensuring compliance with RHPC data management policies.

------------------------
Where to Run Each Script
------------------------

Use the correct node depending on the type of storage:

- **Run on rhea** for:

  - Home directories (``/home``)
  - Project directories (``/projects``)
  - ACL modifications for home or project folders

- **Run on kronos** for:

  - Data directories (``/data``)
  - ACL modifications for archive/derived folders

---------------------------
Scripts and examples
---------------------------

Create home directory
=========================

Creates a home directory at ``/home/<user>`` with private access.

**Run on:** ``rhea``

**Usage:**
::

    sudo create-home -u <username>

**Options:**

+----------------------------+-----------+--------------------------------------------------+
| Parameter                  | Required? | Description                                      |
+============================+===========+==================================================+
| :code:`-u`                 | Yes       | Username for the home dir                        |
+----------------------------+-----------+--------------------------------------------------+
| :code:`-h`, :code:`--help` | No        | Show help message and exit                       |
+----------------------------+-----------+--------------------------------------------------+

- Initializes the user's personal space
- Sets appropriate ownership and permissions
- Access is private (ACL: owner only)

Create project folder
=======================

Creates a ZFS-backed project directory in ``/project-pool/projects/``.

**Run on:** ``rhea``

**Usage:**
::

    sudo create-project -p projectname -q quota -u owner [-a additional_users] [-m permission_mask]

**Options:**

+----------------------------+-----------+-------------------------------------------------------------+
| Parameter                  | Required? | Description                                                 |
+============================+===========+=============================================================+
| :code:`-p`                 | Yes       | Project name                                                |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-q`                 | Yes       | Storage quota (e.g., 100G)                                  |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-u`                 | Yes       | Project owner username                                      |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-a`                 | No        | Comma-separated list of additional users to add ACL entries |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-m`                 | No        | Permission mask for additional users (default: ``rwx``)     |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-h`, :code:`--help` | No        | Show help message and exit                                  |
+----------------------------+-----------+-------------------------------------------------------------+


**Examples:**

Create a project with the required parameters:

::

    sudo create-project -p projectX -q 10G -u alice

Create a project with additional ACL entries for extra users with read and execute permission:

::

    sudo create-project -p projectY -q 20G -u bob -a alice,carol -m r-x

Deleting project folders
========================

Deletes a ZFS project dataset from `/project-pool/projects/`.

**Run on:** `rhea`

**Usage:**
::

    sudo delete-project -p <project_name> [--force]

**Options:**

+------------------------------+-----------+--------------------------------------------------+
| Parameter                    | Required? | Description                                      |
+==============================+===========+==================================================+
| :code:`-p`, :code:`--project`| Yes       | Project name to delete                           |
+------------------------------+-----------+--------------------------------------------------+
| :code:`-f`, :code:`--force`  | No        | Skip confirmation prompt                         |
+------------------------------+-----------+--------------------------------------------------+
| :code:`-h`, :code:`--help`   | No        | Show help message and exit                       |
+------------------------------+-----------+--------------------------------------------------+


Creating data directories
=========================

Creates paired `archive/` and `derived/` directories under `/data-pool/groups/`, either private or public.

**Run on:** `kronos`

**Usage:**

To create a new data directory with the appropriate permissions and ACLs, use:

::

    create-data-dir -u username -g {radiology|aiforoncology} -d directoryname [-a user1,user2] [-m permission_mask]

**Options:**

+----------------------------+-----------+-------------------------------------------------------------+
| Parameter                  | Required? | Description                                                 |
+============================+===========+=============================================================+
| :code:`-u`                 | Yes       | Username of the owner                                       |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-d`                 | Yes       | Name of the directory to create                             |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-g`                 | Yes       | Group name (either ``radiology`` or ``aiforoncology``)      |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-a`                 | No        | Comma-separated list of additional users to grant access    |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-m`                 | No        | Permission mask (default: ``rwx`` if not provided)          |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-h`, :code:`--help` | No        | Show help message and exit                                  |
+----------------------------+-----------+-------------------------------------------------------------+


**Examples:**

Create a private data directory named ``projectX`` for user ``alice`` under ``radiology`` with default permissions:

::

    sudo create-data-dir -u alice -g radiology -d projectX

Create a directory with specific access for ``bob`` and ``charlie``:

::

    sudo create-data-dir -u alice -g aiforoncology -d projectY -a bob,charlie

Create a directory with a custom permission mask:

::

    sudo create-data-dir -u alice -g radiology -d projectZ -m r-x

Deleting data directories
==========================

**Usage:**

To delete a data directory and its corresponding counterpart directories, use:

::

    sudo delete-data-dir -d <path_to_directory> [-f]

**Options:**

+----------------------------+-----------+-------------------------------------------------------------+
| Parameter                  | Required? | Description                                                 |
+============================+===========+=============================================================+
| :code:`-d`                 | Yes       | Absolute or relative path to the directory                  |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-f`                 | No        | Force delete without confirmation                           |
+----------------------------+-----------+-------------------------------------------------------------+
| :code:`-h`, :code:`--help` | No        | Show help message and exit                                  |
+----------------------------+-----------+-------------------------------------------------------------+


**Examples:**

Delete a directory with confirmation:

::

    sudo delete-data-dir -d /data-pool/groups/beets-tan/archive/projectX

Force delete a directory without confirmation:

::

    sudo delete-data-dir -d /data-pool/groups/aiforoncology/archive/projectY -f

Modifying ACLs
==================

**Run on:** ``kronos`` or ``rhea``

**Usage:**

To modify ACLs for directories, use:

::

    sudo modify-acl [OPTIONS]

**Options:**

+-----------------------------+-----------+-----------------------------------------------------------------------+
| Parameter                   | Required? | Description                                                           |
+=============================+===========+=======================================================================+
| :code:`-d`                  | Yes       | Set the target directory                                              |
+-----------------------------+-----------+-----------------------------------------------------------------------+
| :code:`-a`                  | No        | Add users (comma-separated) with specific permissions                 |
+-----------------------------+-----------+-----------------------------------------------------------------------+
| :code:`-r`                  | No        | Remove users (comma-separated) from ACL                               |
+-----------------------------+-----------+-----------------------------------------------------------------------+
| :code:`-m`                  | No        | Specify the permission mask (e.g., ``rwx``)                           |
+-----------------------------+-----------+-----------------------------------------------------------------------+
| :code:`-c`                  | No        | Check ACL consistency across counterpart directories                  |
+-----------------------------+-----------+-----------------------------------------------------------------------+
| :code:`--no-recursive`      | No        | Apply ACL changes without recursion                                   |
+-----------------------------+-----------+-----------------------------------------------------------------------+
| :code:`-h`, :code:`--help`  | No        | Show help message and exit                                            |
+-----------------------------+-----------+-----------------------------------------------------------------------+





**Examples:**

Add users ``david`` and ``eva`` with full permissions to ``projectX``:

::

    sudo modify-acl -d /data-pool/groups/beets-tan/archive/projectX -a david,eva -m rwx

Remove user ``frank`` from ACL of ``projectY``:

::

    sudo modify-acl -d /data-pool/groups/aiforoncology/archive/projectY -r frank

Modify ACL without recursion:

::

    sudo modify-acl -d /data-pool/groups/beets-tan/archive/projectZ -a george -m r-- --no-recursive

Check ACL consistency across archive and derived directories:

::

    sudo modify-acl -d /data-pool/groups/beets-tan/archive/projectX -c

**Note:** If the directory is within ``archive`` or ``derived``, ACL modifications also apply to its counterpart.
