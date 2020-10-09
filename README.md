# ezrsync 0.60

* Repo: http://gitlab.com/pepa65/ezrsync
* Requirements: bash[3.0+: array index expansion: ${!array[@]}]
rsync ln mv rm date find grep [cpio]
(ln provides 1 service, can be removed;
grep could be eliminated with some work;
find, date, mv, rm and rsync are fairly essential;
cpio is optional, mandatory if directive 'cpio' has 1 as argument)
* Copyright: 2012-2016 by dagurasuforge, 2016-2020 pepa65 (sourceforge.net ids)
* License: GNU GPLv3+  <http://gnu.org/licenses/gpl.html>

## Overview
An rsnapshot-like tool which uses fixed time-stamped directories instead of
rotations. The initial author dagurasuforge has seen many rsnapshot-like
scripts/programs, mostly based on Mike Rubels blog, with no real updates to the
idea other than added usability features. Thus most use sync/rotate/promote
with complex crontab structures that require careful timing and use of BOTH
cron AND anacron to deal with hourly backups as well as daily, weekly, etc. on
a system with downtimes. Using only cron, systems could miss daily, weekly,
monthly or yearly backups entirely. Without using cron, hourly backups can't be
done well. (Mike Rubel's original method was the most efficent way to explain
how people could get something working using available tools and minimal work,
but if we're writing a program, we can do better.)

So ezrsync delivers precisely the same result (multiple intervals of retention)
with only one standard call from cron that does all within one cycle (typically
hourly, but it could be longer or shorter), without the need for anacron. There
is no need for renaming, rotating, or promoting any backups after they are
made, but the required number of daily, weekly, monthly and yearly backups are
garanteed to be retained.

## Use
1. Prepare a configuration-file patterned after rsnapshot:
  - Use hourly, daily, weekly, monthly or yearly as interval names, or define other ones.
  - Use only source/destination backup definitions (extra per-backup options will be ignored).
  - All non-implemented or unapplicaple directives will be flagged but ignored.
  - The `snapshot_root`/`destdir` combinations of different configurations must not be the same directory, unless all interval names are different.
2. Either place the config in `/etc/ezrsync.conf` or in `/etc/ezrsync.conf.d/`
or run this script as: `ezrsync --mainconf <configfile>` or
`ezrsync --soleconf <configfile>`
  - If there are scripts present in '/etc/ezrsync.conf.d/' directory, they are
  (also) all run. This directory can only be changed in the main configuration
  file, either in the default one or in the one given as a commandline option
  after `--mainconf`.
  - When `--soleconf` is used, only the specified config file is used, and none of the scripts in any directory.
  - The `lockfile` and `stop_on_stale_lockfile` directives can only be used in
  either the sole or the main configuration-file, and not in any of the files
  in a configuration directory (because there will only be one logfle and
  lockfile for the whole run).
  - The `confdir` directive (to specify an alternate configuration directory)
  can only be set in the main configuration file. If a sole configuration file is specified, it will be the only one used.
  - If a main configuration file is present at the default location or
  specified, it will be used first, and then all the configuration files that
  are present in the default or specified configuration directory.
  - For each configuration file, all unspecified directives will be reset to
  default.
3. Run ezrsync in a single cron entry, no arguments required, like this (for
example every 15 minutes): `*/15 * * * *  root  /usr/local/bin/ezrsync`
It should be run at the smallest interval that is used (or more often), but it
won't run if another instance is still running. If the cron-interval is bigger
than the smallest interval, chances are the larger intervals never get any
backups. (A `crontab.d` script could be packaged with ezrsync without needing
user intervention, except if the smallest used interval is smaller than the
crontab that is packaged.)
4. If the source is empty, the backup will not proceed, so backups will not be
erased.
So if you use sources that are mounted, make sure the mount-directories are
empty when not mounted!
A good idea is to use `chattr +i` when unmounted to make them unwritable even
to root, because files tend to get accidentally written to unmounted mount
points. Backups will over time be replaced by the files that are present in
unmounted source directories. Even if this happens once, the hard links will be
broken, causing more storage space to be used on the destinations!

## Missing features compared to rsnapshot
The `config_version` directive is ignored.
- All `cmd_` directives are ignored except `cmd_rsync`, `cmd_preexec` and
`cmd_postexec` (ssh, logger, du, rsnapshot-diff are not used, nor any
perl-builtin).
- LVM is not explixitly supported, so `linux_lvm_` directives are ignored.
- The `ssh_args` directive is ignored (because ssh-support is not built in, but
it should just work).
- The `du_args` directive is ignored (because du is not used).
- The `one_fs` directive is ignored, so `-x` should be given as an rsync
argument if desired.
- The `include`, `exclude`, `include_file` and `exclude_file` directives are
ignored, so they should be added as rsync arguments if they are needed.
- The `sync_first` directive is ignored (as there is no rotation, `sync_first`
always happens).
- The `lazy_deletes` directive is ignored (they happen by default).
- Per-backup arguments are ignored, so only 2 arguments for the `backup` directive are used.
- Multiple configuration-files can be used instead.
- No support for the `backup_script` directive.

## Special features
1. Compatibility with the rsnapshot configuration-file:
  - They can be processed, but might not fully work as expected unless
  attention is paid to the Missing features above. Both `retain` and `interval`
  directives are understood, and the final argument can be the definition of
  the interval (required for interval names other than hourly, daily, weekly,
  monthly and yearly).
  - These rsnapshot directives are recognised and implemented: `snapshot_root`,   `no_create_root`, `cmd_rsync`, `cmd_preexec`, `cmd_postexec`,
  `retain`/`interval`, `verbose`, `loglevel`, `logfile`, `lockfile`,
  `stop_on_stale_lockfile`, `rsync_short_args`, `rsync_long_args`, `link_dest`
  (which by default uses `find|cpio`) and `backup`.
  - In addition, the following directives are implemented:
    - `timestamplog` (takes 0/1) to specify whether the logfile needs to be
    time-stamped.
    - `confdir` to specify a configuration-file-directory different from the
    default.
    - `rsync_tries` (takes a positive integer) to specify how many times the
    rsync-command needs to be tried.
    - `cpio` (takes 0/1) to specify whether `cpio` or `cp -al` is used.
    - `reuse_trash` (takes 0/1) to use expired backups as a rsync target in
    order to skip prelinking. (Note: while `rsync --link-dest` does keep the
    permissions history, it does so at the expense of not hard-linking
    otherwise identical files. So in case all file permisions get changed,
    backing up using `--link-dest` will copy all files instead of hard-linking
    them!)
  - Directives and arguments are seperated by tabs. All directives have 1
  argument, except:
    - retain/interval: `<interval> <retain> [<definition>]` (if no definition,
    <interval> must be standard)
    - backup: `<source-directory> <destination-directory>`
  - There can only be more than one of the `interval`/`retain` and `backup`
  directives. If either of these is not present at all, nothing will happen.
  - The `snapshot_root` directive is the only directive that is mandatory, all
  others have defined default values.
2. The script can be run more often than the smallest backup cycle that is
used,or less often (in the latter case the larger interval backups might never
happen). No worrying about cron jobs interfering with one another.
3. No anacron is needed. If the computer is off for a period, required backups
will be made at the next run if/when appropriate, and as many backups for each
interval will be kept as specified.
4. The directory-names for ezrsync carry both an explicit time stamp and the
interval name. In case (for example) yearly backups were made when the files
were a mess that day, they can just be deleted, and new ones will be made on
next run. The most recent backup is linked to by the symlink `$symlinkname`
(set to `MostRecentBackup` by default).
5. Forced backups are supported with the `--forced` option followed by an
interval name. They do not count towards the total number of backups to be kept
for that particular interval. They will eventually get pushed out by regular
backups for that interval when there are a sufficient number of newer regular
backups. The regular backups are always properly spaced in time. So in case
(for example) `retain hourly 10`, and 10 hourly backups are forced in one hour,
the old hourly backups will still be kept and slowly be replaced by newer
regular hourly backups, and eventually the forced backups will be deleted when
they get older than the oldest of the regular 10.
6. Independent, persistent backups can be made with the `--independent` option.
These never get automatically deleted and do not interact with the deletion
schedule of other interval backups at all.
7. In case of interruptions, there is a roll forward or roll back mechanism in
place to guard against deletions.
8. Empty source directories are never backed up. This is really a minimal
check, but stricter criteria like tagging backup-sources with hidden files
won't work for read-only sources and may just prevent backups.
9. Different interval specifications can be applied to different sets of backup
directives by using separate configfiles.

## Potential concerns
A date-based system is vulnerable to missed backups or auto-deletion if there
is a clock error. Much consideration has been given to this issue in the
details of the logic. If any backups exists 'in the future', the program is
aborted. But a clock jump to the past does not threaten existing backups
anyway, it just won't make new ones, so has to be detected. If the clock jumps
forward (for example 2 months, perhaps the machine had just been switched off),
things are fine. Your backups, while suddenly being older, will not all
suddenly get expired, because expiry is based on count within an interval, and
not based on time.

## Implementation
For all intervals, starting with the shortest interval:
- if due: make the backups
- if retain fulfilled: remove the obsolete (possibly forced) backups
- if retain not yet fulfilled, look to use trashed backups ("optional
rotation"). In principle, the prelink+rsync happens only once per run at most.
