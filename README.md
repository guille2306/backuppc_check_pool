# Description

This is a helper Bash script to verify the checksum of files in a [BackupPC V4](https://backuppc.github.io/backuppc/) installation.
The latest version of the script can be found at [github.com/guille2306/backuppc_check_pool](https://github.com/guille2306/backuppc_check_pool).

## The problem

On BackupPC V4 and later the files are saved in the pool using their full-file MD5 hash as filename.
Using the [_--checksum_ argument](https://backuppc.github.io/backuppc/BackupPC.html#_conf_rsyncfullargsextra_) (typically during full backups) causes the client to send the full-file checksum for every file.
On the server, _rsync_bpc_ will skip any files that have a matching full-file checksum and metadata (size, mtime and number of hardlinks).

Since both the metadata and the full-file checksum are stored and easily accessed without needing to look at the file contents, the server load is very low.
However, there is a catch: as it relies on the saved checksums, BackupPC will not check again unmodified pool files in the server.
This risks it missing errors due to file system corruption or bit rot in the backup files.

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
The script in this repository has some advantages with respect to it:

- it can be used independently of the nightly job, and even when BackupPC is completely off-line

- by using _zlib-flate_ or _pigz_ it can be much faster to check the cpool

- it traverses the pool in a systematic and deterministic way

It also has some disadvantages:

- it only checks files in the cpool (_TODO_: add uncompressed pool)

- it needs some extra configuration and installation to work

- it's not official ;)

# Use

To start a check run the script as the _backuppc_ user, ideally with a cron job.
The default configuration options are located in the _default.conf_ file.
To override these options you can include them in a _user.conf_ file located in the same folder.

- **bpc_bin_dir =** is the bin folder for BackupPC, where BackupPC_zcat is located

- **dir_base =** is the base folder for the cpool files.
The script will check the files located at "$dir_base/cpool"

- **check_method =** selects the method to use for the standard check.
Possible values are zlib, pigz, and zcat.
Possible errors encountered when using zlib and pigz will be re-checked using BackupPC_zcat.

- **dir_file =** is an external file that saves the last sub-folder of the cpool that was checked by the script.
It must be initialized before the first run, usually to "00".

- **dir_run =** is the number of sub-folders the script will check during each run.
By default the script checks 4 sub-folders per run, so it takes 32 days to sequentially check the full pool if run once a day.

# Thanks

To Alexander Kobel that originally [gave me the idea](https://sourceforge.net/p/backuppc/mailman/message/36379588/) of the script.
To Craig Barratt for the great piece of software that is [BackupPC](https://backuppc.github.io/backuppc/).

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