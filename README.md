# Description

This is a helper Bash script to verify the checksum of files in a [BackupPC V4](https://backuppc.github.io/backuppc/) installation.

## The problem

On BackupPC V4 and later the files are saved in the pool using their full-file MD5 hash
as filename.
Using the [_--checksum_ argument](https://backuppc.github.io/backuppc/BackupPC.html#_conf_rsyncfullargsextra_) (typically during full backups) causes the client to send the full-file checksum for every file.
On the server, _rsync_bpc_ will skip any files that have a matching full-file checksum and metadata (size, mtime and number of hardlinks).

The server load is very low, since both the metadata and the full-file checksum are stored and easily accessed without needing to look at the file contents at all.
However, there is a catch: if you rely on the checksums saved in the server, BackupPC will not check again unmodified pool files.
This risks it missing file system corruption or bit rot in the backup files.

Two possible solutions to verify the integrity of the pool are:

- put the pool in a file system with checksum verification included (like ZFS or Btrfs)

- periodically traverse the pool and checksum all the files

## This script

This script works by comparing the full-file MD5 checksum of the cpool files to their filename, flagging any difference.

The checksum is done uncompressing the files in the cpool using _zlib-flate_, but this can be changed to _pigz_ or _BackupPC_zcat_.
On a severely CPU-limited server (Banana Pi Pro) both _pigz_ and _zlib-flate_ are much faster than _BackupPC_zcat_, they take around a quarter of the time to check the files (_pigz_ is marginally faster than _zlib-flate_).
On the other hand, _BackupPC_zcat_ puts the lowest load on the CPU, _zlib-flate_'s load is 30-35% higher, and _pigz_'s is a whooping 80-100% higher.

However, as _BackupPC_zcat_ produces slightly modified gzip files, there is a (very) small chance that a _BackupPC_zcat_-compressed file will not be properly uncompressed by the other two.
Because of this, the script re-checks every _zlib-flate_ or _pigz_ failure with _BackupPC_zcat_ before calling it a real error.
This produces the best balance between load on the system and time spent checking the pool (at least for a CPU-limited server).

Since BackupPC 4.4.0 there is a [_PoolNightlyDigestCheckPercent_](https://backuppc.github.io/backuppc/BackupPC.html#General-server-configuration) option that does basically the same job during the _BackupPC_nightly_ job, checking a small random percentage of the pool.
The script in this repository has some advantages with respecto to it:

- it can be used independently of the nightly job, and even when BackupPC is completely off-line

- by using _zlib-flate_ or _pigz_ it can be much faster to check the cpool

- it traverses the pool in a systematic and deterministic way

It also has some disadvantages:

- it only checks files in the cpool (_TODO_: add uncompressed pool)

- it needs some extra configuration and installation to work

- it's not official ;)

# Use

To start a check simply run the script as a user that has read-access to the cpool folder.
The configuration is simple:

- the last checked sub-folder of the cpool is saved in the _dir_file_ auxiliary file. This file must be initialized (usually to "00") before the first run.

- by default the script checks 4 sub-folders per run, so it takes 32 days to sequentially check the full pool. This can be modified by changing the external
_for_ loop.

# Thanks

To Alexander Kobel that originally [gave me the idea](https://sourceforge.net/p/backuppc/mailman/message/36379588/) of the script. To Craig Barratt for the great piece of software that is [BackupPC](https://backuppc.github.io/backuppc/).

# Copyright

Copyright (C) 2020 Guillermo Rozas.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program (see the COPYING file). If not, see
[www.gnu.org/licenses](https://www.gnu.org/licenses/).