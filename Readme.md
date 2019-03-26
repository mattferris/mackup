mackup
======

mackup is a flexible backup scheduler.

Usage
-----

To configure a backup schedule, create a directory where your backups will be
stored. This directory is known as the backup *set*.

    # mkdir /var/backups/mackup

Inside the *set* directory, create *targets* (directories) for the days you
want to run your backups.

    # cd /var/backups/mackup
    # mkdir mon

When mackup is run, it will check to see what day it is. On Monday, it will
run the backup configured in the `mon` directory. Now that mackup knows when
to run the backup, we need to tell it how and what to backup.

Each target must contain a subdirectory called `control`. The `control`
directory must contain two files: `run` and `sources`. `sources` contains a
list of file paths (one per line) to backup, while `run` is an exectuable file that actually
performs the backup.

Configure the `mon` target to create a tarball of the `/etc` directory.

First, setup the sources file.

    # echo /etc > mon/control/sources

Next we need to create the `run` file. mackup passes the backup *source path* as
an argument to the run script.  The *source path* is the path defined in the
`sources` file. mackup will change directory to the targets `data/` directory
before executing the run script.

    # cat <<EOF > mon/control/run
    #!/bin/sh

    # source directory
    source="$(dirname $1)/$(basename $1)"

    tar cf "$source.tar" "$(basename $source)".tar
    EOF

Make sure to make the `run` script executable.

    # chmod +x mon/control/run

That's it! All that's left to do is run mackup.

    # mackup /var/backups/mackup

Once mackup is finished, you're backed up data can be found in
`/var/backups/mon/data/etc.tar`. mackup automatically creates the `data`
directory if it doesn't exist.

Configure mackup to run from cron and you have an automatic backup! 

Targets
-------

When mackup runs, it determines a list of targets to search for in the backup
set. The target list will always include the `daily` target, the day name
(`mon` to `sun`), and the day of the month (`1`, `2`, `3`, etc...). If it's a
Monday, then the `weekly` target is also included. Likewise, on the first day
of the month, the `monthly` target is included as well as one of `jan` to `dec`.
Finally, on the first day of the year, the `annually` target is included.

* `daily` - included everyday
* `weekly` - included on Mondays
* `monthly` - included on the first day of each month
* `annually` - included on the first day of each year
* `mon` to `sun` - included on a given day of the week
* `1` to `31` - included on a given day of the month
* `jan` to `dec` - included on the first day of a given month

Scheduling backups to run more often then daily can be accomplished by
scheduling mackup to run as often as you need via cron. Keep in mind, however,
that subsequent backups for a given day may overwrite previous ones unless
the `run` script is careful to write to uniquely identified files or folders
in `data/`.

The `control/` directory
------------------------

Each tagets contains a `control` subdirectory. This directoy contains the `run`
and `sources` files. There are three other directories that can be created
`control` to further refine the behaviour of the backup.

### `control/pre.d/`

The `pre.d` directory contains scripts that are run before each source is
processed. The provides the ability to shutdown services or virtualized guests
before backing up their data.

Additionally, *prerun* scripts can control whether a source should be skipped,
and even if the rest of the sources for the target should be skipped. This is
done by having the *prerun* script exit with codes 255 (skip the source) or
254 (skip the target).

Skipping a source is useful if the *prerun* script was unable to perform a
shutdown of a service or virtualized guest. Skipping a target is useful if
the first source on a remote host fails for connectivity reasons, as the
remaining sources are unlikely to succeed either.

### `control/post.d/`

Similar to `pre.d`, the `post.d` directory contains scripts that are run after
each source is processed. The provides the ability to start services or
virtualized guests up after their data has been backed up.

### `control/env/`

The `env` directory contains a list of files which define environment variables
to be passed to the `run`, *prerun*, and *postrun* scripts. For example,
creating the file `control/env/FOO` with the contents `bar` will allow the `run`
script to access and environment variable `$FOO` with the value `bar`. This
allows `run` scripts to be written more generally, with each target customizing
the scripts behaviour.

Suggested setup
---------------

While mackup doesn't impose much regarding how it's setup on a system, the
following section outlines a suggestion on how to do so (and the source files
are laid out in a way that reflects this suggestion, assuming a Debian file
system standard).

Sets are stored in `/var/lib/mackup`, with logging stored in `/var/log/mackup`.
Sets are enabled by linking them in `/etc/mackup/sets.d`. The cron script in
`/etc/cron.daily/mackup` scans `/etc/mackup/sets.d` for enabled sets. If a set
contains a `log` subdirectoy (preferably a symlink to `/var/log/mackup/<set>`,
then output produced during the backup will be logged to a file, otherwise it
is captured by cron and emailed to the user (in most cases).

Helper scripts
--------------

A number of helper scripts are included in `/usr/share/mackup`. This includes
flexible run scripts for `rsync`, `tar`, and `git` backups, as well as some
service management helpers (for managing service states). These helpers scripts
understand a number of environment variables, which can be use to customize
some of their behaviour (set via `control/env/` for each target).

### run/rsync

Backup sources using `rsync`. Symlink into `control/run` to use.

- `$INCREMENTAL_DAILY` - if exists, link the current day to the previous day using `--link-dest`
- `$INCREMENTAL_MONTHLY` - if exists, link the current month to the previous month using `--list-dest`
- '$INCREMENTAL_NUM' - the number of previous monthly backups to keep (deprecated)
- `$REMOTE_HOST` - address or hostname where sources reside
- '$ROTATE' - the number of previous monthly backups to keep
- `$RSYNC_OPTS` - custom list of options (overrides default of `-av`)
- `$RSYNC_DELETE` - if exists, use the `--delete` option
- `$RSYNC_EXCLUDE_FROM` - specifies a path to use with `--exclude-from`

### run/tar

Backup sources using `tar`. Symlink into `control/run` to use.

- `$REMOTE_HOST` - address or hostname where sources reside
- `$TAR_OPTS` - custom list of options (overrides default of `-v`, `-c` is always used)
- `$TAR_BZIP2` - if exists, bzip the archive (`-j`)
- `$TAR_GZIP` - if exists, gzip the archive (`-z`)

### run/git

Backup sources using `git`. Can cause issues if the sources contain git repos,
as git will consider them to be submodules. Symlink into `control/run` to use.

- `$REMOTE_HOST` - address or hostname where sources reside

### run/zfs

Backup sources using ZFS snapshots (only available for ZFS datasets and volumes).
Symlink into `control/run` to use. 

- `$ROTATE` - the number of previous snapshots to keep
- `$ZFS_OPTS` - custom list of options

### util/remotecmd

Execute a command via SSH on a remote host.

Usage: `# remotecmd <host> <cmd> [ <arg>, <arg>, ... ]`

- `<host>` - the host to run the command on
- `<cmd>` - the path to the command
- `<arg>` - option arguments

To use, drop a script into `control/pre.d` or `control/post.d` with the
following:

    #!/bin/sh
    exec /usr/share/mackup/util/remotecmd example.com virsh shutdown foo

### util/servicectl

Manage a systemd service via `systemctl`. Accepted actions are `is-enabled`,
`status`, `start`, `stop`, and `restart`. The service can reside on the local
system, or on a remote system.

Usage: `# servicectl <host> <service> <action>`

- `<host>` - the host the services resides on (`localhost` for local services)
- `<service>` - the name of the service
- `<action>` - on of: `is-enabled, `status`, `start`, `stop, or `restart`

To use, drop a script into `control/pre.d` or `control/post.d` with the
following:

    #!/bin/sh
    exec /usr/share/mackup/util/servicectl localhost mysql stop

### util/servicectl-helper

Used to easily add *prerun* and *postrun* scripts for managing local services. Just
symlink into `pre.d` or `post.d` and name the link `<action>-<service>`. For
example:

    # ln -s /usr/share/mackup/util/servicectl-helper pre.d/stop-mysql
    # ln -s /usr/share/mackup/util/servicectl-helper post.d/start-mysql

The `servicectl-helper` script checks the name is was executed as - in this
case `stop-mysql` and `start-mysql`.
