# Complete Linux Command Reference

---

## 1. FILE MANAGEMENT

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `ls` | List directory contents | `-l` long format, `-a` show hidden, `-h` human-readable sizes, `-R` recursive, `-S` sort by size, `-t` sort by time, `-r` reverse order, `-i` show inode |
| `cd` | Change current directory | `~` home, `-` previous dir, `..` parent dir |
| `pwd` | Print current working directory | `-P` physical path (resolve symlinks), `-L` logical path |
| `cp` | Copy files and directories | `-r` recursive, `-i` interactive (prompt before overwrite), `-v` verbose, `-u` update (copy only newer), `-p` preserve permissions, `-a` archive mode, `-f` force |
| `mv` | Move or rename files and directories | `-i` interactive, `-v` verbose, `-u` update, `-f` force, `-n` no overwrite |
| `rm` | Remove files or directories | `-r` recursive, `-f` force, `-i` interactive, `-v` verbose, `-d` remove empty dirs |
| `mkdir` | Create directories | `-p` create parent dirs, `-v` verbose, `-m` set permissions |
| `rmdir` | Remove empty directories | `-p` remove parent dirs, `-v` verbose |
| `touch` | Create empty file or update timestamps | `-a` access time only, `-m` modify time only, `-t` set specific time, `-d` parse date string, `-r` reference file |
| `file` | Determine file type | `-b` brief output, `-i` MIME type, `-z` look inside compressed, `-L` follow symlinks |
| `stat` | Display detailed file/filesystem status | `-f` filesystem status, `-c` custom format, `-t` terse output |
| `rename` | Rename multiple files using pattern | `-v` verbose, `-n` dry run, `-f` force |
| `basename` | Strip directory and suffix from filename | `-a` support multiple args, `-s` remove suffix |
| `dirname` | Strip last component from filename | `-z` zero-terminated output |
| `realpath` | Print resolved absolute pathname | `-e` all components must exist, `-m` no requirement on existence, `-s` no symlink resolution |
| `readlink` | Print resolved symlink or canonical path | `-f` canonicalize, `-e` all must exist, `-m` no existence needed |
| `install` | Copy files and set attributes | `-d` create dirs, `-m` set mode, `-o` set owner, `-g` set group, `-v` verbose |
| `shred` | Securely overwrite file to hide contents | `-n` number of overwrites, `-z` add final zero overwrite, `-u` remove after shredding, `-v` verbose |
| `truncate` | Shrink or extend file to specified size | `-s` set size, `-c` don't create file, `-r` reference file |
| `mktemp` | Create temporary file or directory | `-d` create directory, `-p` specify parent dir, `-t` use template, `--suffix` add suffix |

---

## 2. FILE VIEWING & EDITING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `cat` | Concatenate and display file contents | `-n` number lines, `-b` number non-blank lines, `-s` squeeze blank lines, `-E` show line ends, `-T` show tabs, `-A` show all |
| `tac` | Display file in reverse (last line first) | `-s` use separator, `-r` regex separator, `-b` attach separator before |
| `less` | View file contents with backward navigation | `-N` show line numbers, `-S` chop long lines, `-i` ignore case search, `-F` quit if one screen, `-X` no init/deinit |
| `more` | View file contents page by page | `-d` display help, `-s` squeeze blanks, `-n` lines per screen, `+/pattern` start at pattern |
| `head` | Display first lines of a file | `-n` number of lines, `-c` number of bytes, `-q` quiet (no headers) |
| `tail` | Display last lines of a file | `-n` number of lines, `-f` follow (live updates), `-F` follow with retry, `-c` number of bytes, `--pid` terminate with PID |
| `nl` | Number lines of files | `-b` body numbering style, `-s` separator string, `-w` line number width, `-n` number format |
| `wc` | Count lines, words, and bytes in files | `-l` lines, `-w` words, `-c` bytes, `-m` characters, `-L` max line length |
| `sort` | Sort lines of text files | `-r` reverse, `-n` numeric sort, `-k` sort by field, `-t` field delimiter, `-u` unique, `-h` human-numeric, `-f` ignore case, `-o` output file |
| `uniq` | Report or omit repeated lines | `-c` count occurrences, `-d` only duplicates, `-u` only unique, `-i` ignore case, `-f` skip fields, `-s` skip chars |
| `cut` | Remove sections from each line | `-d` delimiter, `-f` fields, `-c` characters, `-b` bytes, `--complement` invert selection |
| `paste` | Merge lines of files side by side | `-d` delimiter, `-s` serial (one file per line) |
| `tr` | Translate or delete characters | `-d` delete chars, `-s` squeeze repeats, `-c` complement set, `[:upper:]` `[:lower:]` `[:digit:]` classes |
| `rev` | Reverse lines character by character | (no major options) |
| `fold` | Wrap lines to specified width | `-w` width, `-s` break at spaces, `-b` count bytes |
| `fmt` | Simple text formatter | `-w` max line width, `-s` split only, `-u` uniform spacing |
| `column` | Format input into columns | `-t` create table, `-s` delimiter, `-n` disable merging, `-c` column width |
| `expand` | Convert tabs to spaces | `-t` tab stops, `-i` initial tabs only |
| `unexpand` | Convert spaces to tabs | `-t` tab stops, `-a` convert all, `--first-only` only leading |
| `colrm` | Remove columns from lines | (positional args: start col, end col) |
| `pr` | Convert text files for printing | `-d` double space, `-h` header, `-l` page length, `-o` margin, `-w` page width |
| `nano` | Simple terminal text editor | `-B` backup, `-c` show cursor position, `-i` auto-indent, `-l` line numbers, `-m` mouse support |
| `vi` / `vim` | Advanced terminal text editor | `-R` read-only, `-b` binary mode, `-d` diff mode, `-O` vertical split, `-o` horizontal split, `+n` start at line n |
| `emacs` | Extensible text editor | `-nw` terminal mode, `-q` no init file, `--batch` batch mode, `-f` run function |
| `sed` | Stream editor for text transformation | `-i` in-place edit, `-e` expression, `-n` suppress output, `-r`/`-E` extended regex, `-f` script file |
| `awk` | Pattern scanning and text processing language | `-F` field separator, `-v` assign variable, `-f` program file, `-i` in-place |
| `diff` | Compare files line by line | `-u` unified format, `-r` recursive, `-i` ignore case, `-w` ignore whitespace, `-y` side-by-side, `-q` brief, `--color` colorize |
| `diff3` | Compare three files line by line | `-m` merge, `-e` ed script, `-A` show all changes |
| `sdiff` | Side-by-side file comparison and merge | `-w` width, `-o` output file, `-s` suppress common lines |
| `patch` | Apply diff patches to files | `-p` strip prefix, `-R` reverse, `-b` backup, `--dry-run` test without applying, `-d` change directory |
| `cmp` | Compare two files byte by byte | `-b` print differing bytes, `-l` list all differences, `-s` silent, `-n` compare first N bytes |
| `comm` | Compare two sorted files line by line | `-1` suppress col 1, `-2` suppress col 2, `-3` suppress col 3, `--check-order` verify sorted |
| `vimdiff` | Edit two or three files and show differences | (same as vim options) |
| `strings` | Print printable strings from binary files | `-a` scan entire file, `-n` min string length, `-t` print offset, `-e` encoding |
| `od` | Dump files in octal and other formats | `-A` address radix, `-t` output format, `-x` hex, `-c` chars, `-N` max bytes |
| `xxd` | Create hex dump or reverse hex dump | `-r` reverse (hex to binary), `-l` length, `-s` seek, `-g` group size, `-c` columns, `-p` plain hex |
| `hexdump` | Display file in hex, decimal, octal, or ASCII | `-C` canonical hex+ASCII, `-n` length, `-s` skip, `-v` no squeezing |

---

## 3. FILE SEARCH & FIND

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `find` | Search for files in directory hierarchy | `-name` by name, `-iname` case-insensitive, `-type` (f/d/l), `-size`, `-mtime` modified time, `-exec` run command, `-perm` permissions, `-user`, `-group`, `-maxdepth`, `-mindepth`, `-delete`, `-empty`, `-newer` |
| `locate` | Find files by name using database | `-i` ignore case, `-c` count, `-l` limit results, `-r` regex, `-e` check existence |
| `updatedb` | Update the locate database | `--prunepaths` exclude dirs, `--output` db file |
| `whereis` | Locate binary, source, and man page for command | `-b` binaries only, `-m` manuals only, `-s` sources only |
| `which` | Show full path of shell commands | `-a` show all matches |
| `type` | Show how a command name is interpreted | `-a` all locations, `-t` type only (alias/function/builtin/file) |
| `grep` | Search text using patterns | `-i` ignore case, `-r` recursive, `-n` line numbers, `-v` invert match, `-l` files with matches, `-c` count, `-w` whole word, `-E` extended regex, `-P` perl regex, `-o` only matching, `-A` after context, `-B` before context, `-C` context, `--include` file pattern, `--exclude` file pattern, `--color` |
| `egrep` | Extended regex grep (same as `grep -E`) | (same as grep) |
| `fgrep` | Fixed string grep (same as `grep -F`) | (same as grep) |
| `rgrep` | Recursive grep (same as `grep -r`) | (same as grep) |
| `ag` | The Silver Searcher — fast code search | `-i` ignore case, `-l` files only, `-c` count, `--hidden` search hidden, `-G` file pattern, `-w` whole word |
| `ripgrep` (`rg`) | Ultra-fast regex search tool | `-i` ignore case, `-n` line numbers, `-l` files only, `-t` file type, `-g` glob, `--hidden`, `-w` word, `-c` count, `-z` search compressed |

---

## 4. FILE PERMISSIONS & OWNERSHIP

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `chmod` | Change file permissions | `-R` recursive, `-v` verbose, `-c` report changes, `u/g/o/a` user/group/other/all, `+/-/=` add/remove/set, `r/w/x` read/write/execute, octal (e.g., 755) |
| `chown` | Change file owner and group | `-R` recursive, `-v` verbose, `-c` report changes, `--reference` copy from file, `-h` affect symlink |
| `chgrp` | Change file group ownership | `-R` recursive, `-v` verbose, `-c` report changes, `--reference` copy from file |
| `umask` | Set default permission mask for new files | `-S` symbolic output, `-p` print in reusable form |
| `getfacl` | Get file Access Control Lists | `-R` recursive, `-a` access ACL, `-d` default ACL, `-p` absolute paths |
| `setfacl` | Set file Access Control Lists | `-m` modify, `-x` remove, `-R` recursive, `-b` remove all, `-d` default ACL, `-k` remove default |
| `chattr` | Change file attributes on ext filesystem | `+i` immutable, `+a` append only, `+s` secure deletion, `+u` undeletable, `-R` recursive |
| `lsattr` | List file attributes on ext filesystem | `-R` recursive, `-a` all files, `-d` directories only |

---

## 5. FILE COMPRESSION & ARCHIVING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `tar` | Archive files into a tarball | `-c` create, `-x` extract, `-t` list, `-v` verbose, `-f` file, `-z` gzip, `-j` bzip2, `-J` xz, `--exclude` pattern, `-C` change dir, `-p` preserve permissions, `--strip-components` |
| `gzip` | Compress files with gzip | `-d` decompress, `-k` keep original, `-v` verbose, `-r` recursive, `-1` to `-9` compression level, `-l` list info |
| `gunzip` | Decompress gzip files | `-k` keep original, `-v` verbose, `-f` force, `-r` recursive |
| `bzip2` | Compress files with bzip2 | `-d` decompress, `-k` keep original, `-v` verbose, `-1` to `-9` compression level |
| `bunzip2` | Decompress bzip2 files | `-k` keep original, `-v` verbose, `-f` force |
| `xz` | Compress files with xz | `-d` decompress, `-k` keep original, `-v` verbose, `-0` to `-9` compression level, `-T` threads |
| `unxz` | Decompress xz files | `-k` keep original, `-v` verbose |
| `zip` | Create zip archives | `-r` recursive, `-e` encrypt, `-x` exclude, `-u` update, `-d` delete from archive, `-9` max compression, `-q` quiet |
| `unzip` | Extract zip archives | `-d` destination dir, `-l` list, `-o` overwrite, `-n` never overwrite, `-q` quiet, `-P` password |
| `zcat` | View compressed file contents without extracting | (same as `gzip -dc`) |
| `zless` | View compressed files with pager | (same as less options) |
| `zgrep` | Search inside compressed files | (same as grep options) |
| `bzcat` | View bzip2 compressed file contents | (no major options) |
| `xzcat` | View xz compressed file contents | (no major options) |
| `7z` | 7-Zip archiver with high compression | `a` add, `x` extract, `l` list, `t` test, `-p` password, `-mmt` multithread |
| `rar` | Create RAR archives | `a` add, `x` extract, `l` list, `-p` password, `-r` recursive |
| `unrar` | Extract RAR archives | `x` extract with path, `e` extract flat, `l` list, `t` test, `-p` password |
| `compress` | Compress files using LZW | `-v` verbose, `-f` force, `-r` recursive |
| `uncompress` | Decompress .Z files | `-v` verbose, `-f` force |
| `zstd` | Compress/decompress with Zstandard | `-d` decompress, `-k` keep, `-v` verbose, `-1` to `-19` level, `-T` threads, `--rm` remove source |
| `lz4` | Extremely fast compression | `-d` decompress, `-k` keep, `-v` verbose, `-1` to `-12` level |
| `cpio` | Copy files to/from archives | `-o` create, `-i` extract, `-t` list, `-d` create dirs, `-v` verbose, `-p` pass-through |
| `ar` | Create/modify archive files (static libraries) | `r` insert, `x` extract, `t` list, `d` delete, `s` add index |

---

## 6. DISK & STORAGE MANAGEMENT

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `df` | Report filesystem disk space usage | `-h` human-readable, `-T` show filesystem type, `-i` inode info, `-a` all filesystems, `--total` grand total |
| `du` | Estimate file/directory space usage | `-h` human-readable, `-s` summary, `-a` all files, `-c` grand total, `--max-depth` depth limit, `--exclude` pattern, `-k` kilobytes |
| `fdisk` | Partition table manipulator | `-l` list partitions, `-u` display in sectors, `-s` partition size |
| `gdisk` | GPT partition table manipulator | `-l` list partitions |
| `parted` | Partition manipulator (GPT & MBR) | `-l` list, `-s` script mode, `mkpart` create, `resizepart` resize, `rm` remove |
| `lsblk` | List block devices | `-f` filesystem info, `-a` all devices, `-o` select columns, `-p` full path, `-t` topology |
| `blkid` | Locate/print block device attributes | `-o` output format, `-s` specific tag, `-p` low-level probing |
| `mount` | Mount a filesystem | `-t` type, `-o` options (ro/rw/noexec/nosuid), `-a` mount all in fstab, `-r` read-only, `--bind` bind mount, `-l` list |
| `umount` | Unmount a filesystem | `-l` lazy unmount, `-f` force, `-a` unmount all, `-R` recursive |
| `mkfs` | Create a filesystem | `-t` type (ext4/xfs/btrfs), `-L` label, `-V` verbose |
| `mkfs.ext4` | Create ext4 filesystem | `-L` label, `-b` block size, `-i` bytes-per-inode, `-j` journal, `-m` reserved blocks % |
| `mkfs.xfs` | Create XFS filesystem | `-L` label, `-b` block size, `-f` force, `-d` data section |
| `mkfs.btrfs` | Create Btrfs filesystem | `-L` label, `-f` force, `-d` data profile, `-m` metadata profile |
| `mkfs.vfat` | Create FAT filesystem | `-F` FAT size (12/16/32), `-n` label, `-s` sectors per cluster |
| `mkswap` | Set up a Linux swap area | `-L` label, `-f` force |
| `swapon` | Enable swap space | `-a` all, `-s` summary, `-p` priority, `--show` display |
| `swapoff` | Disable swap space | `-a` all |
| `fsck` | Filesystem consistency check | `-t` type, `-a` auto-repair, `-y` yes to all, `-n` no changes, `-f` force check |
| `e2fsck` | Check ext2/3/4 filesystem | `-p` auto-repair, `-y` yes to all, `-f` force, `-n` read-only |
| `xfs_repair` | Repair XFS filesystem | `-n` no modify (check only), `-v` verbose, `-L` zero log |
| `tune2fs` | Adjust ext filesystem parameters | `-l` list superblock, `-c` max mount count, `-i` check interval, `-L` label, `-m` reserved blocks %, `-O` features |
| `resize2fs` | Resize ext2/3/4 filesystem | `-p` progress, `-f` force, `-M` minimum size |
| `xfs_growfs` | Expand XFS filesystem | `-d` grow data, `-l` grow log, `-D` data size |
| `lvm` | Logical Volume Manager commands | (see pvcreate, vgcreate, lvcreate below) |
| `pvcreate` | Initialize physical volume for LVM | `-f` force, `-v` verbose |
| `pvdisplay` | Display physical volume info | `-v` verbose, `-m` map |
| `pvs` | Report physical volume info (brief) | `-o` fields, `-a` all |
| `vgcreate` | Create a volume group | `-s` PE size |
| `vgdisplay` | Display volume group info | `-v` verbose |
| `vgs` | Report volume group info (brief) | `-o` fields |
| `lvcreate` | Create a logical volume | `-L` size, `-n` name, `-l` extents, `--type` (linear/striped/mirror) |
| `lvdisplay` | Display logical volume info | `-v` verbose, `-m` map |
| `lvs` | Report logical volume info (brief) | `-o` fields |
| `lvextend` | Extend logical volume size | `-L` size, `-l` extents, `-r` resize filesystem |
| `lvreduce` | Reduce logical volume size | `-L` size, `-f` force, `-r` resize filesystem |
| `lsscsi` | List SCSI devices | `-g` generic devices, `-l` long output, `-s` show size |
| `hdparm` | Get/set SATA/IDE device parameters | `-i` device info, `-t` read timing, `-T` cache timing, `-S` standby, `-C` power mode |
| `smartctl` | Control SMART monitoring on disks | `-a` all info, `-H` health, `-t` run test, `-l` logs, `-i` device info |
| `dd` | Convert and copy a file (low-level) | `if=` input file, `of=` output file, `bs=` block size, `count=` blocks, `status=progress`, `conv=` conversions |
| `sync` | Flush filesystem buffers to disk | `-f` sync one file, `-d` data only |
| `fstrim` | Discard unused blocks on mounted filesystem | `-a` all, `-v` verbose, `-o` offset, `-l` length |
| `quota` | Display disk usage and limits | `-u` user, `-g` group, `-v` verbose, `-s` human-readable |
| `quotaon` | Turn on filesystem quotas | `-a` all, `-u` user, `-g` group |
| `quotaoff` | Turn off filesystem quotas | `-a` all, `-u` user, `-g` group |
| `edquota` | Edit user/group quotas | `-u` user, `-g` group, `-t` grace period, `-p` prototype |
| `repquota` | Report filesystem quotas | `-a` all, `-u` user, `-g` group, `-s` human-readable |

---

## 7. LINKS

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `ln` | Create hard and symbolic links | `-s` symbolic link, `-f` force, `-v` verbose, `-i` interactive, `-r` relative, `-b` backup |
| `unlink` | Remove a single file (simpler than rm) | (no major options) |
| `readlink` | Print value of a symbolic link | `-f` canonicalize, `-e` all must exist, `-m` no existence required |
| `symlinks` | Find and manage symbolic links | `-c` convert absolute to relative, `-d` delete dangling, `-r` recursive, `-t` test/report |

---

## 8. USER & GROUP MANAGEMENT

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `useradd` | Create a new user | `-m` create home dir, `-d` home dir path, `-s` shell, `-g` primary group, `-G` supplementary groups, `-u` UID, `-e` expiry date, `-c` comment, `-r` system account |
| `userdel` | Delete a user account | `-r` remove home dir, `-f` force |
| `usermod` | Modify a user account | `-aG` append to groups, `-d` home dir, `-l` new username, `-s` shell, `-L` lock, `-U` unlock, `-e` expiry, `-g` primary group |
| `adduser` | Interactive user creation (Debian) | `--home` home dir, `--shell` shell, `--ingroup` group, `--disabled-password`, `--system` |
| `deluser` | Interactive user deletion (Debian) | `--remove-home`, `--remove-all-files`, `--backup`, `--group` |
| `groupadd` | Create a new group | `-g` GID, `-r` system group, `-f` force |
| `groupdel` | Delete a group | `-f` force |
| `groupmod` | Modify a group | `-n` new name, `-g` new GID |
| `groups` | Show group memberships of a user | (no major options) |
| `id` | Display user and group IDs | `-u` user ID, `-g` group ID, `-G` all groups, `-n` name instead of number |
| `whoami` | Print current username | (no major options) |
| `who` | Show who is logged on | `-a` all, `-b` boot time, `-H` headers, `-q` count |
| `w` | Show who is logged in and what they're doing | `-h` no header, `-s` short format, `-f` show from field |
| `last` | Show last logged-in users | `-n` number of entries, `-a` display hostname last, `-x` show shutdown/reboot, `-F` full times |
| `lastb` | Show failed login attempts | `-n` number, `-a` hostname last |
| `lastlog` | Report most recent login of all users | `-u` specific user, `-t` days |
| `passwd` | Change user password | `-l` lock, `-u` unlock, `-d` delete password, `-e` expire (force change), `-S` status, `-n` min days, `-x` max days |
| `chpasswd` | Update passwords in batch mode | `-e` encrypted passwords, `-m` MD5 |
| `chage` | Change user password aging info | `-l` list, `-d` last change, `-E` expiry date, `-M` max days, `-m` min days, `-W` warning days |
| `su` | Switch user identity | `-` full login shell, `-c` command, `-s` shell, `-l` login shell |
| `sudo` | Execute command as another user (root) | `-u` user, `-i` login shell, `-s` shell, `-l` list permissions, `-k` invalidate timestamp, `-v` validate, `-e` edit (sudoedit), `-b` background |
| `visudo` | Safely edit the sudoers file | `-f` file, `-c` check syntax, `-s` strict |
| `finger` | User information lookup | `-l` long format, `-s` short format |
| `chfn` | Change user finger information | `-f` full name, `-r` room, `-w` work phone, `-h` home phone |
| `chsh` | Change user login shell | `-s` shell, `-l` list shells |
| `newgrp` | Log in to a new group | (no major options) |
| `sg` | Execute command as different group | (no major options) |
| `gpasswd` | Administer /etc/group | `-a` add user, `-d` remove user, `-A` set admins, `-M` set members, `-r` remove password |
| `pwck` | Verify integrity of password files | `-r` read-only, `-s` sort |
| `grpck` | Verify integrity of group files | `-r` read-only, `-s` sort |
| `nologin` | Deny login to an account | (used as shell: `/sbin/nologin`) |
| `faillog` | Display/set login failure limits | `-u` user, `-m` max failures, `-r` reset, `-a` all users |

---

## 9. PROCESS MANAGEMENT

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `ps` | Report snapshot of current processes | `aux` all processes (BSD), `-ef` all processes (POSIX), `-p` specific PID, `--sort` sort by field, `-o` custom output, `-u` by user, `--forest` tree view |
| `top` | Real-time process monitoring | `-d` delay, `-p` specific PID, `-u` user, `-n` iterations, `-b` batch mode, `-H` threads |
| `htop` | Interactive process viewer (enhanced top) | `-d` delay, `-u` user, `-p` PID, `-s` sort column, `-t` tree mode |
| `kill` | Send signal to a process | `-9` SIGKILL, `-15` SIGTERM, `-1` SIGHUP, `-l` list signals, `-s` signal name |
| `killall` | Kill processes by name | `-9` SIGKILL, `-i` interactive, `-u` by user, `-v` verbose, `-w` wait, `-e` exact name |
| `pkill` | Kill processes by pattern | `-f` match full command, `-u` by user, `-signal` specify signal, `-x` exact match, `-n` newest, `-o` oldest |
| `pgrep` | Find process IDs by pattern | `-l` list name, `-f` match full command, `-u` by user, `-a` full command, `-x` exact, `-n` newest, `-o` oldest, `-c` count |
| `pidof` | Find PID of a running program | `-s` single PID, `-x` scripts too |
| `nice` | Run command with modified scheduling priority | `-n` adjustment value (-20 highest to 19 lowest) |
| `renice` | Alter priority of running processes | `-n` priority value, `-p` PID, `-u` user, `-g` group |
| `nohup` | Run command immune to hangups | (redirect output with `> file 2>&1 &`) |
| `disown` | Remove job from shell job table | `-h` mark so SIGHUP not sent, `-a` all jobs, `-r` running jobs |
| `bg` | Resume suspended job in background | `%n` job number |
| `fg` | Bring background job to foreground | `%n` job number |
| `jobs` | List active jobs in current shell | `-l` show PIDs, `-n` only changed, `-r` running only, `-s` stopped only |
| `wait` | Wait for background processes to finish | `PID` specific process, `-n` any process, `-f` wait for termination |
| `timeout` | Run command with time limit | `-s` signal, `-k` kill after, `--preserve-status`, `--foreground` |
| `watch` | Execute command periodically and show output | `-n` interval seconds, `-d` highlight changes, `-t` no title, `-g` exit on change, `-c` color |
| `at` | Schedule a one-time command | `-l` list jobs (atq), `-d` delete job (atrm), `-f` read from file, `-m` mail on completion |
| `atq` | List pending at jobs | (no major options) |
| `atrm` | Remove at jobs | (specify job number) |
| `batch` | Execute commands when system load permits | (same as at) |
| `crontab` | Schedule recurring commands | `-e` edit, `-l` list, `-r` remove, `-u` user |
| `cron` | Daemon for executing scheduled commands | (runs as service) |
| `anacron` | Run delayed cron jobs after missed schedules | `-f` force, `-u` update only, `-s` serialize, `-T` test config |
| `systemctl` | Control systemd services and system | `start`, `stop`, `restart`, `reload`, `status`, `enable`, `disable`, `is-active`, `is-enabled`, `list-units`, `list-unit-files`, `daemon-reload`, `mask`, `unmask`, `--type` unit type |
| `service` | SysVinit service management | `start`, `stop`, `restart`, `status`, `--status-all` |
| `chkconfig` | Enable/disable SysVinit services | `--list`, `--level`, `on/off` |
| `init` | Process control initialization | `0` halt, `1` single user, `3` multi-user, `5` graphical, `6` reboot |
| `telinit` | Change SysVinit runlevel | (same as init) |
| `runlevel` | Show current and previous runlevel | (no major options) |
| `pstree` | Display process tree | `-p` show PIDs, `-u` show user, `-a` show arguments, `-h` highlight current, `-l` long lines |
| `strace` | Trace system calls and signals | `-p` attach to PID, `-f` follow forks, `-e` filter calls, `-o` output file, `-c` summary, `-t` timestamps |
| `ltrace` | Trace library calls | `-p` attach to PID, `-f` follow forks, `-e` filter, `-o` output file, `-c` summary |
| `lsof` | List open files and network connections | `-p` by PID, `-u` by user, `-i` network, `-c` by command, `+D` by directory, `-t` PID only |
| `fuser` | Identify processes using files/sockets | `-k` kill processes, `-m` mounted filesystem, `-v` verbose, `-n` namespace (tcp/udp) |
| `uptime` | Show system uptime and load averages | `-p` pretty format, `-s` since boot |
| `time` | Measure command execution time | `-v` verbose (GNU), `-p` POSIX format |
| `times` | Show accumulated process times | (built-in, no options) |
| `ionice` | Set/get I/O scheduling class and priority | `-c` class (0-3), `-n` priority (0-7), `-p` PID |
| `taskset` | Set/get CPU affinity | `-p` PID, `-c` CPU list |
| `chrt` | Set/get real-time scheduling attributes | `-p` PID, `-f` FIFO, `-r` round-robin, `-o` other, `-m` show min/max priorities |
| `nproc` | Print number of available processors | `--all` all installed |
| `mpstat` | Report CPU statistics | `-P` specific CPU, `interval` seconds, `count` |
| `pidstat` | Report statistics for Linux tasks | `-u` CPU, `-r` memory, `-d` I/O, `-p` PID, `-t` threads |
| `schedtool` | Query and set CPU scheduling parameters | `-e` execute with policy, `-a` affinity, `-n` nice |

---

## 10. SYSTEM INFORMATION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `uname` | Print system information | `-a` all, `-r` kernel release, `-s` kernel name, `-m` machine arch, `-n` hostname, `-v` kernel version, `-p` processor, `-o` OS |
| `hostname` | Show or set system hostname | `-f` FQDN, `-i` IP address, `-I` all IPs, `-d` domain, `-s` short name |
| `hostnamectl` | Control system hostname (systemd) | `set-hostname`, `status`, `--static`, `--transient`, `--pretty` |
| `arch` | Print machine architecture | (no major options) |
| `lscpu` | Display CPU architecture information | `-e` extended, `-p` parseable, `-J` JSON |
| `lshw` | List detailed hardware information | `-short` brief, `-class` category, `-html` HTML output, `-json` JSON |
| `lspci` | List PCI devices | `-v` verbose, `-vv` very verbose, `-k` kernel drivers, `-nn` numeric+names, `-t` tree |
| `lsusb` | List USB devices | `-v` verbose, `-t` tree, `-s` specific device |
| `lsmod` | Show loaded kernel modules | (no major options) |
| `modinfo` | Show kernel module information | `-p` parameters, `-d` description, `-a` author, `-n` filename |
| `modprobe` | Add/remove kernel modules | `-r` remove, `-v` verbose, `--show-depends` dependencies, `-n` dry run, `-c` show config |
| `insmod` | Insert a kernel module (low-level) | (specify module file path) |
| `rmmod` | Remove a kernel module (low-level) | `-f` force, `-v` verbose |
| `depmod` | Generate module dependency info | `-a` all modules, `-n` dry run, `-v` verbose |
| `dmesg` | Print kernel ring buffer messages | `-l` log level, `-T` human-readable time, `-w` follow, `-c` clear, `-H` human-readable, `--level` filter level |
| `dmidecode` | DMI/SMBIOS hardware info from BIOS | `-t` type (bios/system/baseboard/processor/memory), `-s` keyword |
| `free` | Display memory usage | `-h` human-readable, `-m` megabytes, `-g` gigabytes, `-t` total line, `-s` repeat interval, `-c` count |
| `vmstat` | Virtual memory statistics | `interval` seconds, `count`, `-s` event counter, `-d` disk stats, `-w` wide output, `-S` unit |
| `iostat` | CPU and I/O statistics | `-c` CPU only, `-d` device only, `-x` extended, `-h` human-readable, `-p` partitions, `interval` `count` |
| `sar` | Collect and report system activity | `-u` CPU, `-r` memory, `-b` I/O, `-d` disk, `-n` network, `-q` queue, `-f` read from file |
| `sysctl` | Read/write kernel parameters | `-a` display all, `-w` write value, `-p` load from file, `-n` no key names |
| `procinfo` | Display system status from /proc | `-a` all info, `-d` show delta, `-n` repeat count |
| `nproc` | Show number of processing units | `--all` all installed CPUs |
| `getconf` | Display system configuration values | `_NPROCESSORS_ONLN`, `LONG_BIT`, `PAGE_SIZE` |
| `timedatectl` | Control system time and date (systemd) | `status`, `set-time`, `set-timezone`, `list-timezones`, `set-ntp` |
| `date` | Display or set system date/time | `-u` UTC, `-d` display given string, `+FORMAT` custom format, `-s` set date, `-R` RFC 2822 |
| `cal` | Display calendar | `-3` three months, `-y` full year, `-m` start Monday, `-j` Julian day |
| `hwclock` | Read/set hardware clock | `--show`, `--set`, `--systohc` system→hardware, `--hctosys` hardware→system, `--utc` |
| `locale` | Display locale information | `-a` all locales, `-m` all charmaps |
| `localectl` | Control system locale settings | `status`, `set-locale`, `list-locales`, `set-keymap` |
| `printenv` | Print environment variables | (specify variable name) |
| `env` | Display/modify environment for a command | `-i` ignore inherited, `-u` unset variable |
| `set` | Display or set shell options/variables | `-e` exit on error, `-x` debug/trace, `-u` error on unset vars, `-o` option name |
| `export` | Set environment variables for child processes | `-n` remove export, `-p` list all exports |
| `lsb_release` | Display Linux distribution info | `-a` all, `-i` distributor, `-d` description, `-r` release, `-c` codename |
| `cat /etc/os-release` | Show OS identification data | (file, not command) |
| `hostid` | Print numeric host identifier | (no major options) |
| `logger` | Send messages to system log | `-p` priority, `-t` tag, `-i` PID, `-s` stderr too, `-f` file |

---

## 11. NETWORKING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `ip` | Show/manipulate routing, devices, tunnels | `addr` addresses, `link` devices, `route` routes, `neigh` ARP, `-s` stats, `-c` color, `add/del/show` |
| `ifconfig` | Configure network interfaces (deprecated) | `-a` all interfaces, `up/down` enable/disable, `netmask`, `broadcast` |
| `ping` | Send ICMP echo requests to hosts | `-c` count, `-i` interval, `-s` packet size, `-t` TTL, `-w` deadline, `-W` timeout, `-q` quiet, `-f` flood |
| `ping6` | Send ICMPv6 echo requests | (same as ping for IPv6) |
| `traceroute` | Trace packet route to host | `-n` no DNS, `-m` max hops, `-w` timeout, `-q` queries per hop, `-I` ICMP, `-T` TCP |
| `tracepath` | Trace path to host (no root needed) | `-n` no DNS, `-b` print both name and IP, `-l` initial packet length |
| `mtr` | Combined traceroute and ping tool | `-r` report mode, `-c` count, `-n` no DNS, `-w` wide report, `-i` interval |
| `netstat` | Network statistics (deprecated, use ss) | `-t` TCP, `-u` UDP, `-l` listening, `-p` PID, `-n` numeric, `-a` all, `-r` routing, `-s` statistics, `-i` interfaces |
| `ss` | Socket statistics (replacement for netstat) | `-t` TCP, `-u` UDP, `-l` listening, `-p` processes, `-n` numeric, `-a` all, `-s` summary, `-4` IPv4, `-6` IPv6, `-o` timers |
| `nslookup` | Query DNS nameservers interactively | `-type=` record type (A/MX/NS/CNAME/SOA), `server` DNS server |
| `dig` | DNS lookup utility | `+short` brief, `+trace` recursive trace, `-t` type, `@server` DNS server, `+noall +answer` clean output, `ANY` all records |
| `host` | Simple DNS lookup utility | `-t` record type, `-a` all records, `-v` verbose, `-4` IPv4, `-6` IPv6 |
| `nmap` | Network exploration and security scanner | `-sS` SYN scan, `-sT` TCP scan, `-sU` UDP scan, `-p` ports, `-O` OS detect, `-sV` version detect, `-A` aggressive, `-Pn` skip ping, `-oN` output, `-sn` ping sweep |
| `curl` | Transfer data from/to URLs | `-o` output file, `-O` save with original name, `-L` follow redirect, `-d` POST data, `-H` header, `-X` method, `-u` auth, `-k` insecure SSL, `-s` silent, `-v` verbose, `-I` head only, `-b` cookie, `-c` cookie jar, `-F` form upload |
| `wget` | Download files from web | `-O` output file, `-c` resume, `-r` recursive, `-b` background, `-q` quiet, `-P` prefix dir, `-np` no parent, `--limit-rate`, `--mirror`, `--user`/`--password` |
| `ssh` | Secure Shell remote login | `-p` port, `-i` identity file, `-L` local port forward, `-R` remote port forward, `-D` SOCKS proxy, `-X` X11 forward, `-N` no command, `-f` background, `-v` verbose, `-o` option, `-J` jump host |
| `scp` | Secure copy files over SSH | `-P` port, `-r` recursive, `-i` identity file, `-p` preserve times, `-C` compression, `-q` quiet, `-l` bandwidth limit |
| `sftp` | Secure FTP over SSH | `-P` port, `-i` identity file, `-b` batch file, `-r` recursive |
| `rsync` | Remote (and local) file synchronization | `-a` archive, `-v` verbose, `-z` compress, `-r` recursive, `-P` progress+partial, `--delete` remove extra files, `-e` remote shell, `-n` dry run, `--exclude`, `--include` |
| `ftp` | File Transfer Protocol client | `-p` passive, `-n` no auto-login, `-v` verbose, `-i` no interactive prompt |
| `telnet` | Telnet client (insecure) | (specify host and port) |
| `nc` / `netcat` | TCP/UDP network tool (Swiss army knife) | `-l` listen, `-p` port, `-v` verbose, `-z` scan, `-u` UDP, `-w` timeout, `-e` execute, `-k` keep listening |
| `socat` | Multipurpose relay for bidirectional data | `TCP-LISTEN:port`, `TCP:host:port`, `STDIO`, `FILE:`, `EXEC:`, `-d` debug |
| `tcpdump` | Capture and analyze network packets | `-i` interface, `-n` no DNS, `-c` count, `-w` write to file, `-r` read from file, `-X` hex+ASCII, `-A` ASCII, `-s` snap length, `-v` verbose, `port`, `host` filters |
| `tshark` | Terminal-based Wireshark (packet analysis) | `-i` interface, `-f` capture filter, `-Y` display filter, `-c` count, `-w` output file, `-r` read file, `-V` verbose |
| `iptables` | IPv4 firewall rule management | `-A` append, `-D` delete, `-I` insert, `-L` list, `-F` flush, `-P` policy, `-s` source, `-d` dest, `-p` protocol, `-j` target, `--dport`, `--sport`, `-m` match |
| `ip6tables` | IPv6 firewall rule management | (same as iptables) |
| `nftables` / `nft` | Modern firewall framework (replaces iptables) | `list ruleset`, `add rule`, `add table`, `add chain`, `delete`, `flush` |
| `firewall-cmd` | Firewalld command-line tool | `--add-port`, `--add-service`, `--remove-port`, `--list-all`, `--permanent`, `--reload`, `--zone`, `--get-active-zones` |
| `ufw` | Uncomplicated Firewall (Ubuntu) | `enable`, `disable`, `allow`, `deny`, `reject`, `status`, `delete`, `reset`, `--dry-run` |
| `route` | Show/manipulate IP routing table (deprecated) | `-n` numeric, `add`, `del`, `-net`, `-host`, `gw`, `netmask` |
| `arp` | Manipulate ARP cache (deprecated) | `-a` display, `-d` delete, `-s` set static, `-n` numeric |
| `arping` | Send ARP requests to host | `-c` count, `-I` interface, `-f` quit on reply |
| `ethtool` | Query/control network driver and hardware | `-s` change setting, `-i` driver info, `-S` NIC statistics, `-a` pause params, `-k` offload |
| `iwconfig` | Configure wireless network interfaces | `essid`, `mode`, `freq`, `key`, `power` |
| `iwlist` | Display detailed wireless info | `scan`, `frequency`, `rate`, `power` |
| `iw` | Modern wireless device configuration | `dev`, `link`, `scan`, `connect`, `info`, `set` |
| `nmcli` | NetworkManager command-line tool | `device status`, `connection show`, `connection up/down`, `general`, `radio wifi`, `device wifi list` |
| `nmtui` | NetworkManager TUI (text user interface) | (interactive menus) |
| `dhclient` | DHCP client for obtaining IP | `-r` release, `-v` verbose, `-4` IPv4, `-6` IPv6 |
| `whois` | Query WHOIS database for domain info | `-h` server, `-p` port |
| `ab` | Apache HTTP benchmarking tool | `-n` requests, `-c` concurrency, `-t` timelimit, `-H` header, `-p` POST file, `-T` content-type |
| `iperf` / `iperf3` | Network bandwidth measurement | `-s` server mode, `-c` client mode, `-p` port, `-t` duration, `-u` UDP, `-b` bandwidth, `-P` parallel |
| `ncat` | Enhanced netcat from nmap | `-l` listen, `-p` port, `--ssl` encrypt, `-e` execute, `-k` keep open, `--proxy` |
| `bridge` | Show/manipulate bridge devices | `link`, `fdb`, `vlan`, `show`, `add`, `del` |
| `tc` | Traffic control for QoS | `qdisc`, `class`, `filter`, `show`, `add`, `del` |

---

## 12. SSH & KEY MANAGEMENT

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `ssh-keygen` | Generate SSH key pairs | `-t` type (rsa/ed25519/ecdsa), `-b` bits, `-f` filename, `-C` comment, `-N` passphrase, `-p` change passphrase, `-R` remove host |
| `ssh-copy-id` | Install SSH key on remote server | `-i` identity file, `-p` port, `-f` force |
| `ssh-add` | Add private keys to SSH agent | `-l` list fingerprints, `-L` list public keys, `-d` delete key, `-D` delete all, `-t` lifetime |
| `ssh-agent` | Hold private keys for SSH auth | `-s` Bourne shell, `-c` C shell, `-k` kill agent |
| `ssh-keyscan` | Gather SSH public host keys | `-t` key type, `-p` port, `-H` hash output |
| `sshd` | SSH server daemon | `-p` port, `-f` config file, `-D` don't detach, `-d` debug, `-t` test config |
| `ssh-config` | SSH client configuration file | (file at `~/.ssh/config`: Host, HostName, User, Port, IdentityFile, ProxyJump) |

---

## 13. PACKAGE MANAGEMENT

### Debian/Ubuntu (APT/DPKG)

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `apt` | High-level package manager (Debian/Ubuntu) | `install`, `remove`, `update`, `upgrade`, `full-upgrade`, `search`, `show`, `list`, `autoremove`, `purge`, `-y` auto-yes |
| `apt-get` | Legacy APT package tool | `install`, `remove`, `update`, `upgrade`, `dist-upgrade`, `autoremove`, `-y`, `-f` fix broken, `--reinstall`, `--purge`, `-d` download only |
| `apt-cache` | Query APT package cache | `search`, `show`, `showpkg`, `depends`, `rdepends`, `policy`, `stats` |
| `dpkg` | Low-level Debian package manager | `-i` install, `-r` remove, `-P` purge, `-l` list, `-s` status, `-L` list files, `-S` search file owner, `--configure`, `--unpack` |
| `dpkg-reconfigure` | Reconfigure installed packages | `--frontend`, `--priority` |
| `add-apt-repository` | Add PPA/repository to sources | `-y` auto-confirm, `-r` remove, `-u` update |
| `apt-key` | Manage APT repository keys | `add`, `del`, `list`, `export` |
| `apt-file` | Search for files in packages | `search`, `list`, `update` |

### Red Hat/CentOS/Fedora (YUM/DNF/RPM)

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `yum` | Package manager (RHEL/CentOS 7) | `install`, `remove`, `update`, `search`, `info`, `list`, `groupinstall`, `clean`, `-y` auto-yes, `--enablerepo`, `--disablerepo` |
| `dnf` | Next-gen package manager (Fedora/RHEL 8+) | `install`, `remove`, `update`, `upgrade`, `search`, `info`, `list`, `autoremove`, `clean`, `group install`, `-y`, `--refresh` |
| `rpm` | Low-level RPM package manager | `-i` install, `-U` upgrade, `-e` erase, `-q` query, `-qa` all packages, `-ql` list files, `-qi` info, `-qf` which package owns file, `-V` verify, `--import` GPG key |
| `yum-config-manager` | Manage yum repos and config | `--add-repo`, `--enable`, `--disable` |

### Arch Linux (Pacman)

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `pacman` | Package manager for Arch Linux | `-S` install, `-R` remove, `-Syu` update system, `-Ss` search, `-Si` info, `-Q` query installed, `-Ql` list files, `-Sc` clean cache, `-Rs` remove with deps |
| `yay` / `paru` | AUR helper for Arch Linux | (same as pacman options, plus AUR support) |

### Universal/Other

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `snap` | Snap package manager (Ubuntu) | `install`, `remove`, `refresh`, `list`, `find`, `info`, `enable`, `disable`, `--classic`, `--edge` |
| `flatpak` | Flatpak application manager | `install`, `uninstall`, `update`, `list`, `search`, `run`, `info`, `--user`, `--system` |
| `pip` | Python package installer | `install`, `uninstall`, `list`, `freeze`, `show`, `search`, `-r` requirements, `--upgrade`, `--user` |
| `npm` | Node.js package manager | `install`, `uninstall`, `update`, `list`, `search`, `init`, `-g` global, `--save-dev`, `run` |
| `gem` | Ruby package manager | `install`, `uninstall`, `list`, `search`, `update`, `environment`, `-v` version |
| `cargo` | Rust package manager | `build`, `run`, `test`, `install`, `update`, `search`, `--release` |
| `brew` | Homebrew package manager (Linux/macOS) | `install`, `uninstall`, `update`, `upgrade`, `list`, `search`, `info`, `doctor`, `cleanup` |

---

## 14. TEXT PROCESSING & MANIPULATION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `awk` | Pattern scanning and processing language | `-F` field sep, `-v` variable, `-f` program file, `BEGIN/END` blocks, `NR` line number, `NF` field count |
| `sed` | Stream editor for filtering and transforming text | `-i` in-place, `-e` expression, `-n` suppress output, `-E` extended regex, `s/old/new/g` substitute |
| `grep` | Print lines matching a pattern | `-i` ignore case, `-r` recursive, `-n` line numbers, `-v` invert, `-c` count, `-l` files only, `-E` extended regex, `-P` Perl regex |
| `cut` | Remove sections from each line of files | `-d` delimiter, `-f` fields, `-c` characters, `-b` bytes |
| `paste` | Merge lines of files | `-d` delimiter, `-s` serial |
| `tr` | Translate or delete characters | `-d` delete, `-s` squeeze, `-c` complement |
| `sort` | Sort lines of text | `-r` reverse, `-n` numeric, `-k` key field, `-t` delimiter, `-u` unique, `-h` human-numeric |
| `uniq` | Report or filter repeated lines | `-c` count, `-d` duplicates, `-u` unique, `-i` ignore case |
| `tee` | Read from stdin, write to stdout and files | `-a` append, `-i` ignore interrupt |
| `xargs` | Build and execute commands from stdin | `-I{}` replace string, `-n` max args, `-P` parallel, `-d` delimiter, `-0` null delimiter, `-p` prompt, `-t` verbose |
| `printf` | Format and print data | (C-style format: `%s` string, `%d` integer, `%f` float, `\n` newline, `\t` tab) |
| `echo` | Display line of text | `-n` no newline, `-e` enable escapes, `-E` disable escapes |
| `yes` | Output string repeatedly | (specify string, default "y") |
| `seq` | Print numeric sequences | `-s` separator, `-w` equal width, `-f` format |
| `expr` | Evaluate expressions | `+` `-` `*` `/` `%` arithmetic, `:` regex match, `length`, `substr` |
| `bc` | Arbitrary precision calculator | `-l` math library, `-q` quiet, `scale` decimal places |
| `dc` | Reverse-polish desk calculator | (stack-based, `p` print, `q` quit) |
| `factor` | Print prime factors of numbers | (no major options) |
| `jq` | Command-line JSON processor | `.field` access, `[]` array, `-r` raw output, `-c` compact, `-e` exit status, `--arg` variable, `select()` filter |
| `yq` | Command-line YAML/XML processor | `.field` access, `-r` raw, `-i` in-place, `-y` YAML output |
| `xmllint` | XML validation and formatting | `--format` pretty-print, `--xpath` query, `--noout` no output, `--schema` validate, `--html` parse HTML |
| `csvtool` | CSV file manipulation | `col`, `namedcol`, `head`, `drop`, `join`, `-t` delimiter |
| `iconv` | Convert character encoding | `-f` from encoding, `-t` to encoding, `-l` list encodings, `-o` output file |
| `base64` | Base64 encode/decode | `-d` decode, `-w` wrap column |
| `md5sum` | Compute MD5 hash | `-c` check file, `-b` binary, `--quiet` |
| `sha1sum` | Compute SHA-1 hash | `-c` check, `-b` binary |
| `sha256sum` | Compute SHA-256 hash | `-c` check, `-b` binary |
| `sha512sum` | Compute SHA-512 hash | `-c` check, `-b` binary |
| `cksum` | Print CRC checksum and byte count | (no major options) |
| `sum` | Print checksum and block count | `-r` BSD style, `-s` System V style |
| `b2sum` | Compute BLAKE2 hash | `-l` hash length, `-c` check |
| `split` | Split file into pieces | `-b` bytes per file, `-l` lines per file, `-n` number of files, `-d` numeric suffix, `-a` suffix length |
| `csplit` | Split file based on context/pattern | `-f` prefix, `-n` digit count, `-z` remove empty, `-k` keep on error |
| `join` | Join lines with common field | `-t` delimiter, `-1` file1 field, `-2` file2 field, `-a` unpaired, `-e` empty replacement |
| `tac` | Concatenate and print files in reverse | `-s` separator, `-r` regex separator |
| `shuf` | Generate random permutations | `-n` count, `-i` range, `-e` treat args as input, `-o` output file |

---

## 15. SHELL & SCRIPTING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `bash` | GNU Bourne Again Shell | `-c` command string, `-x` debug trace, `-e` exit on error, `-n` syntax check, `-v` verbose, `--login`, `--norc`, `--noprofile` |
| `sh` | POSIX shell | `-c` command, `-x` debug, `-e` exit on error |
| `zsh` | Z Shell | `-l` login shell, `-i` interactive, `-c` command |
| `fish` | Friendly Interactive Shell | `-c` command, `-l` login, `--no-config` |
| `csh` / `tcsh` | C Shell / Enhanced C Shell | `-f` fast start, `-v` verbose, `-x` debug |
| `ksh` | Korn Shell | `-c` command, `-r` restricted |
| `source` / `.` | Execute commands from file in current shell | (specify script file) |
| `exec` | Replace shell with command | (replaces current process) |
| `eval` | Evaluate string as shell command | (evaluate arguments as command) |
| `alias` | Create command shortcuts | `-p` print all aliases |
| `unalias` | Remove command aliases | `-a` remove all |
| `history` | Show command history | `-c` clear, `-d` delete entry, `-w` write to file, `-r` read from file, `!n` execute nth, `!!` repeat last |
| `fc` | Fix/edit and re-execute commands from history | `-l` list, `-s` substitute and execute, `-e` editor |
| `set` | Set/unset shell options and positional params | `-e` exit on error, `-x` trace, `-u` unset error, `-o` option, `-f` disable globbing |
| `unset` | Unset variables and functions | `-v` variable (default), `-f` function |
| `shift` | Shift positional parameters left | (specify count, default 1) |
| `getopts` | Parse positional parameters (shell builtin) | (used in shell scripts for option parsing) |
| `read` | Read line from stdin into variable | `-p` prompt, `-s` silent, `-t` timeout, `-n` char count, `-a` array, `-r` raw (no backslash escape), `-d` delimiter |
| `declare` / `typeset` | Declare variables with attributes | `-i` integer, `-a` array, `-A` associative array, `-r` readonly, `-x` export, `-p` print |
| `local` | Declare local variables in functions | (same as declare but local scope) |
| `readonly` | Make variables read-only | `-a` array, `-f` function, `-p` print |
| `let` | Evaluate arithmetic expression | (e.g., `let "a = 5 + 3"`) |
| `test` / `[` | Evaluate conditional expressions | `-f` is file, `-d` is dir, `-e` exists, `-r`/`-w`/`-x` permissions, `-z` empty string, `-n` non-empty, `-eq`/`-ne`/`-lt`/`-gt` numeric compare |
| `[[` | Enhanced test (bash/zsh) | (same as test plus `=~` regex, `&&` `||` operators) |
| `trap` | Catch signals and execute commands | `EXIT`, `INT`, `TERM`, `HUP`, `ERR`, `DEBUG` |
| `exit` | Exit the shell with status code | (specify return code, default 0) |
| `return` | Return from function with status | (specify return code) |
| `true` | Return success (exit code 0) | (no options) |
| `false` | Return failure (exit code 1) | (no options) |
| `sleep` | Delay for specified time | `s` seconds (default), `m` minutes, `h` hours, `d` days |
| `wait` | Wait for process to complete | `-n` any process, `-f` wait for termination, PID |
| `xdg-open` | Open file/URL with default application | (no major options) |
| `script` | Record terminal session to file | `-a` append, `-c` command, `-q` quiet, `-t` timing file |
| `scriptreplay` | Replay terminal session recording | `--timing` timing file, `--typescript` script file |

---

## 16. REDIRECTION & PIPES

| Syntax | Description |
|--------|-------------|
| `>` | Redirect stdout to file (overwrite) |
| `>>` | Redirect stdout to file (append) |
| `2>` | Redirect stderr to file |
| `2>>` | Redirect stderr to file (append) |
| `&>` or `> file 2>&1` | Redirect both stdout and stderr to file |
| `<` | Redirect stdin from file |
| `<<` | Here document (inline input) |
| `<<<` | Here string |
| `\|` | Pipe stdout to next command's stdin |
| `\|&` | Pipe both stdout and stderr |
| `tee` | Split output to file and stdout |

---

## 17. BOOT & INIT SYSTEM

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `systemctl` | Control systemd system and services | `start`, `stop`, `restart`, `status`, `enable`, `disable`, `is-active`, `list-units`, `daemon-reload`, `mask`, `unmask`, `poweroff`, `reboot`, `suspend`, `hibernate` |
| `journalctl` | Query systemd journal logs | `-u` unit, `-f` follow, `-b` current boot, `-p` priority, `--since`/`--until` time range, `-n` lines, `-k` kernel, `-r` reverse, `--no-pager`, `-o` output format |
| `systemd-analyze` | Analyze boot performance | `blame`, `critical-chain`, `plot`, `time`, `dot` |
| `reboot` | Restart the system | `-f` force, `-p` poweroff, `--no-wall` no message |
| `shutdown` | Halt, power off, or reboot the system | `-h` halt, `-r` reboot, `-c` cancel, `now` immediately, `+n` minutes, `HH:MM` schedule, `--no-wall` |
| `poweroff` | Power off the system | `-f` force, `--no-wall` no broadcast |
| `halt` | Halt the system | `-f` force, `-p` poweroff, `--no-wall` |
| `init` | Process initializer (SysVinit) | `0-6` runlevels, `q` re-read inittab |
| `grub-install` | Install GRUB bootloader | `--target` platform, `--boot-directory`, `--efi-directory`, `--recheck` |
| `grub-mkconfig` | Generate GRUB configuration | `-o` output file |
| `update-grub` | Update GRUB config (Debian shortcut) | (no major options) |
| `dracut` | Generate initramfs image | `-f` force, `--kver` kernel version, `-v` verbose |
| `mkinitramfs` | Create initramfs image (Debian) | `-o` output file |
| `mkinitcpio` | Create initramfs image (Arch) | `-p` preset, `-g` generate, `-k` kernel |

---

## 18. LOGGING & MONITORING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `journalctl` | View systemd journal logs | `-u` unit, `-f` follow, `-b` boot, `-p` priority, `--since`, `--until`, `-n` lines, `-k` kernel |
| `syslog` | System logging protocol/daemon | (config at `/etc/rsyslog.conf`) |
| `rsyslog` | Rocket-fast system log daemon | (config files in `/etc/rsyslog.d/`) |
| `logrotate` | Rotate, compress, and manage log files | `-f` force, `-d` debug/dry-run, `-s` state file, `-v` verbose |
| `logger` | Send messages to system logger | `-p` priority, `-t` tag, `-i` include PID, `-s` also stderr |
| `dmesg` | Print kernel ring buffer messages | `-T` timestamps, `-w` follow, `-l` level, `-H` human-readable, `-c` read and clear |
| `last` | Show listing of last logged-in users | `-n` count, `-a` hostname last, `-x` shutdown/reboot |
| `lastb` | Show bad login attempts | `-n` count, `-a` hostname last |
| `lastlog` | Reports the most recent login | `-u` user, `-t` days |
| `auditctl` | Control Linux audit system | `-l` list rules, `-a` add rule, `-d` delete rule, `-w` watch file, `-p` permissions filter, `-k` key |
| `ausearch` | Search audit logs | `-k` key, `-m` message type, `-ts` start time, `-te` end time, `-i` interpret |
| `aureport` | Produce audit reports | `-au` authentication, `-l` logins, `-f` files, `--summary` |
| `fail2ban-client` | Control fail2ban intrusion prevention | `status`, `set`, `get`, `start`, `stop`, `reload` |

---

## 19. SECURITY & ENCRYPTION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `gpg` | GNU Privacy Guard (encryption/signing) | `--gen-key` generate, `-e` encrypt, `-d` decrypt, `-s` sign, `--verify`, `--import`, `--export`, `-r` recipient, `-a` ASCII armor, `--list-keys`, `--delete-key` |
| `gpg2` | GnuPG version 2 | (same as gpg) |
| `openssl` | OpenSSL cryptographic toolkit | `genrsa`, `req`, `x509`, `enc`, `dgst`, `s_client`, `s_server`, `-in`, `-out`, `-cert`, `-key`, `verify`, `pkcs12` |
| `ssh-keygen` | Generate/manage SSH keys | `-t` type, `-b` bits, `-f` file, `-C` comment, `-N` passphrase |
| `passwd` | Change user password | `-l` lock, `-u` unlock, `-d` delete, `-e` expire |
| `chage` | Change password aging policy | `-l` list, `-M` max days, `-m` min days, `-E` expiry |
| `cryptsetup` | Manage LUKS encrypted volumes | `luksFormat`, `luksOpen`, `luksClose`, `luksDump`, `luksAddKey`, `luksRemoveKey`, `status` |
| `dm-crypt` | Disk encryption subsystem | (used via cryptsetup) |
| `encfs` | Encrypted filesystem in userspace | (interactive setup) |
| `ecryptfs` | Enterprise encrypted filesystem | (kernel-level encryption) |
| `aide` | Advanced Intrusion Detection Environment | `--init`, `--check`, `--update`, `--compare`, `-c` config |
| `tripwire` | File integrity monitoring | `--init`, `--check`, `--update-policy` |
| `chroot` | Run command in different root directory | (specify new root and command) |
| `firejail` | SUID sandbox for restricting apps | `--private`, `--net=`, `--seccomp`, `--caps.drop=all`, `--noprofile` |
| `apparmor_status` | Show AppArmor status | (no major options) |
| `aa-enforce` | Set AppArmor profile to enforce mode | (specify profile) |
| `aa-complain` | Set AppArmor profile to complain mode | (specify profile) |
| `getenforce` | Get SELinux enforcing mode | (no major options) |
| `setenforce` | Set SELinux enforcing mode | `0` permissive, `1` enforcing |
| `sestatus` | Show SELinux status | `-v` verbose |
| `semanage` | SELinux policy management | `port`, `fcontext`, `boolean`, `login`, `user` |
| `restorecon` | Restore SELinux security context | `-R` recursive, `-v` verbose, `-F` force |
| `chcon` | Change SELinux security context of file | `-t` type, `-u` user, `-r` role, `-R` recursive |
| `setsebool` | Set SELinux boolean value | `-P` persistent |
| `getsebool` | Get SELinux boolean value | `-a` all booleans |
| `pam_tally2` | Login failure counter | `--reset`, `--user` |
| `faillock` | Display/reset login failures | `--user`, `--reset`, `--dir` |

---

## 20. CONTAINER & VIRTUALIZATION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `docker` | Docker container management | `run`, `build`, `pull`, `push`, `ps`, `images`, `exec`, `logs`, `stop`, `start`, `rm`, `rmi`, `network`, `volume`, `-d` detach, `-it` interactive, `-p` port, `-v` volume mount, `--name`, `-e` env vars |
| `docker-compose` / `docker compose` | Multi-container Docker orchestration | `up`, `down`, `build`, `start`, `stop`, `restart`, `logs`, `ps`, `exec`, `-d` detach, `-f` file, `--build` |
| `podman` | Daemonless container engine (Docker compatible) | (same as docker commands) |
| `buildah` | Build OCI container images | `from`, `run`, `copy`, `commit`, `push`, `bud` (build-using-dockerfile) |
| `skopeo` | Work with container images and registries | `inspect`, `copy`, `delete`, `list-tags`, `sync` |
| `crictl` | CRI-compatible container runtime CLI | `ps`, `images`, `pull`, `logs`, `exec`, `inspect`, `pods` |
| `kubectl` | Kubernetes cluster management | `get`, `describe`, `create`, `apply`, `delete`, `logs`, `exec`, `port-forward`, `scale`, `-n` namespace, `-o` output format, `--context` |
| `minikube` | Run local Kubernetes cluster | `start`, `stop`, `delete`, `status`, `dashboard`, `service` |
| `helm` | Kubernetes package manager | `install`, `upgrade`, `uninstall`, `list`, `repo add`, `search`, `template`, `rollback` |
| `vagrant` | Build and manage virtual machines | `init`, `up`, `halt`, `destroy`, `ssh`, `status`, `provision`, `reload`, `box` |
| `virsh` | Manage KVM/libvirt virtual machines | `list`, `start`, `shutdown`, `destroy`, `create`, `define`, `undefine`, `console`, `dominfo`, `dumpxml` |
| `virt-install` | Create KVM virtual machines | `--name`, `--ram`, `--vcpus`, `--disk`, `--cdrom`, `--os-variant`, `--network` |
| `qemu` | Generic machine emulator | `-m` memory, `-smp` CPUs, `-hda` disk, `-cdrom`, `-boot`, `-enable-kvm`, `-nographic` |
| `lxc` | Linux container management | `launch`, `list`, `start`, `stop`, `delete`, `exec`, `file`, `config`, `image` |
| `lxd` | LXD container/VM daemon | (managed through lxc client) |
| `systemd-nspawn` | Lightweight container (systemd) | `-D` directory, `-b` boot, `--machine`, `--network-veth` |
| `chroot` | Run command with different root filesystem | (specify new root) |
| `unshare` | Run program with unshared namespaces | `-m` mount, `-u` UTS, `-i` IPC, `-p` PID, `-n` network, `-U` user, `-r` map root |
| `nsenter` | Enter namespaces of another process | `-t` target PID, `-m` mount, `-u` UTS, `-n` network, `-p` PID |
| `cgroups` | Control groups resource management | (managed via `/sys/fs/cgroup/`, `cgcreate`, `cgset`, `cgexec`) |

---

## 21. GIT & VERSION CONTROL

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `git init` | Initialize new git repository | `--bare` bare repo |
| `git clone` | Clone a repository | `--depth` shallow clone, `--branch`, `--recurse-submodules`, `--single-branch` |
| `git add` | Stage changes for commit | `-A` all, `-p` interactive patch, `.` current dir |
| `git commit` | Record changes to repository | `-m` message, `-a` stage tracked files, `--amend` modify last commit, `-s` sign-off, `--no-edit` |
| `git status` | Show working tree status | `-s` short, `-b` branch info, `--ignored` |
| `git log` | Show commit history | `--oneline`, `--graph`, `--all`, `-n` limit, `--author`, `--since`/`--until`, `--stat`, `-p` patches, `--pretty` |
| `git diff` | Show changes between commits/working tree | `--staged`/`--cached`, `--stat`, `--name-only`, `--color-words`, `HEAD~n` |
| `git branch` | List, create, or delete branches | `-d` delete, `-D` force delete, `-a` all, `-r` remote, `-m` rename, `-v` verbose |
| `git checkout` | Switch branches or restore files | `-b` new branch, `-f` force, `--` files only |
| `git switch` | Switch branches (modern) | `-c` create, `-d` detach |
| `git restore` | Restore working tree files (modern) | `--staged` unstage, `--source` commit, `-p` interactive |
| `git merge` | Join branches together | `--no-ff` no fast-forward, `--squash`, `--abort`, `--continue` |
| `git rebase` | Reapply commits on top of another base | `-i` interactive, `--onto`, `--abort`, `--continue`, `--skip` |
| `git pull` | Fetch and integrate remote changes | `--rebase`, `--no-rebase`, `--ff-only`, `--all` |
| `git push` | Update remote refs with local commits | `-u` set upstream, `--force`, `--tags`, `--all`, `--delete` |
| `git fetch` | Download remote objects and refs | `--all`, `--prune`, `--tags`, `--depth` |
| `git remote` | Manage remote repository connections | `add`, `remove`, `rename`, `-v` verbose, `set-url`, `show` |
| `git stash` | Temporarily store uncommitted changes | `push`, `pop`, `list`, `apply`, `drop`, `clear`, `-u` include untracked, `-p` patch |
| `git tag` | Create, list, or delete tags | `-a` annotated, `-m` message, `-d` delete, `-l` list, `-v` verify |
| `git reset` | Reset HEAD to specified state | `--soft`, `--mixed` (default), `--hard`, `HEAD~n` |
| `git revert` | Create commit that undoes a previous commit | `--no-commit`, `--no-edit` |
| `git cherry-pick` | Apply specific commits from another branch | `-n` no commit, `--abort`, `--continue`, `-x` add source reference |
| `git bisect` | Binary search for bug-introducing commit | `start`, `bad`, `good`, `reset`, `run` |
| `git blame` | Show who changed each line of a file | `-L` line range, `-e` show email, `-w` ignore whitespace |
| `git show` | Show various types of objects | `--stat`, `--name-only`, `--format` |
| `git reflog` | Show reference log (history of HEAD) | `--all`, `-n` limit, `--date` |
| `git clean` | Remove untracked files | `-f` force, `-d` directories, `-n` dry run, `-x` include ignored, `-i` interactive |
| `git submodule` | Manage sub-repositories | `add`, `init`, `update`, `status`, `foreach`, `--recursive` |
| `git worktree` | Manage multiple working trees | `add`, `list`, `remove`, `prune` |
| `git archive` | Create archive of files from tree | `--format` (tar/zip), `--prefix`, `-o` output file |
| `git config` | Get and set git configuration | `--global`, `--local`, `--system`, `--list`, `--unset`, `-e` edit |
| `git gc` | Cleanup and optimize repository | `--aggressive`, `--prune`, `--auto` |
| `git fsck` | Verify connectivity and validity of objects | `--full`, `--unreachable`, `--dangling` |

---

## 22. TRANSFER & SYNCHRONIZATION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `rsync` | Fast, versatile file copying/sync tool | `-a` archive, `-v` verbose, `-z` compress, `-P` progress, `--delete`, `-e ssh`, `-n` dry run, `--exclude`, `--bwlimit` |
| `scp` | Secure copy over SSH | `-r` recursive, `-P` port, `-i` key, `-p` preserve, `-C` compress |
| `sftp` | Secure FTP over SSH | `-P` port, `-b` batch, `-r` recursive |
| `wget` | Non-interactive network downloader | `-O` output, `-c` continue, `-r` recursive, `-q` quiet, `--mirror`, `--limit-rate` |
| `curl` | Transfer URL data | `-o` output, `-O` remote name, `-L` follow redirects, `-d` data, `-H` header, `-X` method, `-s` silent |
| `ftp` | FTP client | `-p` passive, `-n` no auto-login |
| `lftp` | Sophisticated FTP/HTTP client | `mirror`, `pget`, `mget`, `--parallel`, `-e` execute |
| `rclone` | Cloud storage sync tool | `copy`, `sync`, `move`, `ls`, `lsd`, `mount`, `config`, `--progress`, `--dry-run` |
| `aria2c` | Multi-protocol download utility | `-x` connections, `-s` split, `-j` concurrent, `-c` continue, `--seed-time`, `-d` dir |
| `axel` | Light download accelerator | `-n` connections, `-o` output, `-a` progress |

---

## 23. PRINTING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `lp` | Print files | `-d` destination, `-n` copies, `-o` options, `-P` page range |
| `lpr` | Print files (BSD style) | `-P` printer, `-#` copies, `-o` options |
| `lpq` | Show printer queue status | `-P` printer, `-a` all printers, `-l` long format |
| `lprm` | Cancel print jobs | `-P` printer, `-` all jobs |
| `lpstat` | Print system status | `-a` accepting, `-p` printers, `-t` all info, `-d` default |
| `cancel` | Cancel print jobs | `-a` all jobs |
| `cupsd` | CUPS printing daemon | `-f` foreground, `-c` config file |
| `cupsctl` | Configure CUPS server | `--remote-admin`, `--share-printers`, `--debug-logging` |
| `lpadmin` | Configure CUPS printers | `-p` printer, `-E` enable, `-v` device URI, `-m` model, `-d` default |
| `a2ps` | Format files for PostScript printing | `-o` output, `-1`/`-2` columns, `-P` printer, `--medium` paper size |
| `enscript` | Convert text to PostScript | `-o` output, `-2` two columns, `-r` landscape, `-G` fancy header |
| `ps2pdf` | Convert PostScript to PDF | (GhostScript options) |

---

## 24. MAIL & MESSAGING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `mail` / `mailx` | Send and receive mail | `-s` subject, `-a` attachment, `-c` CC, `-b` BCC |
| `sendmail` | Send email via SMTP | `-t` read recipients from header, `-f` from address, `-v` verbose |
| `postfix` | Postfix mail server | `start`, `stop`, `reload`, `status`, `check`, `flush` |
| `mutt` | Terminal email client | `-s` subject, `-a` attach, `-c` CC, `-b` BCC, `-i` include file |
| `alpine` | Text-based email client | `-i` go to inbox, `-f` folder, `-c` compose |
| `wall` | Send message to all logged-in users | `-n` suppress banner |
| `write` | Send message to specific user | (specify user and optional tty) |
| `mesg` | Control write access to terminal | `y` allow, `n` deny |
| `talk` | Talk to another user in real-time | (specify user@host) |

---

## 25. ENVIRONMENT & CONFIGURATION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `export` | Set environment variable for child processes | `-n` remove, `-p` list all |
| `env` | Run command with modified environment | `-i` empty env, `-u` unset var |
| `printenv` | Print environment variables | (specify variable name) |
| `set` | Set shell options and variables | `-e` exit on error, `-x` trace, `-u` unset error |
| `unset` | Remove variable or function | `-v` variable, `-f` function |
| `source` / `.` | Execute file in current shell | (specify file) |
| `xdg-mime` | Query/set MIME type associations | `query default`, `default`, `query filetype` |
| `xdg-settings` | Get/set default applications | `get`, `set`, `check` |
| `update-alternatives` | Manage symlinks for default commands | `--install`, `--config`, `--list`, `--remove`, `--display`, `--auto` |
| `stty` | Change/print terminal line settings | `-a` all settings, `sane` reset, `-echo` disable echo, `raw` raw mode, `size` terminal size |
| `tput` | Terminal capability values | `cols`, `lines`, `clear`, `setaf` color, `bold`, `sgr0` reset |
| `reset` | Reset terminal to sane values | (no major options) |
| `clear` | Clear terminal screen | `-x` don't clear scrollback |
| `tty` | Print terminal file name | `-s` silent (exit status only) |
| `dircolors` | Set colors for ls | `-b` Bourne shell, `-c` C shell, `-p` print defaults |
| `loadkeys` | Load keyboard translation tables | (specify keymap file) |
| `setxkbmap` | Set X keyboard layout | `-layout`, `-variant`, `-option`, `-query` |
| `xrandr` | Configure display (X11) | `--output`, `--mode`, `--rate`, `--rotate`, `--pos`, `--off`, `--auto`, `--primary` |
| `xset` | Set X display preferences | `s` screensaver, `dpms` power, `r` repeat rate, `b` bell |
| `xdpyinfo` | Display information about X server | (no major options) |
| `xmodmap` | Modify X keymaps and button mappings | `-e` expression, `-pke` print keymap |

---

## 26. DEVELOPMENT & BUILD TOOLS

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `gcc` | GNU C compiler | `-o` output, `-c` compile only, `-g` debug info, `-O` optimize, `-Wall` all warnings, `-I` include path, `-L` lib path, `-l` link library, `-std` standard, `-shared` shared lib, `-static` |
| `g++` | GNU C++ compiler | (same as gcc for C++) |
| `make` | Build automation tool | `-f` makefile, `-j` parallel jobs, `-n` dry run, `-C` directory, `-B` unconditional, `target` |
| `cmake` | Cross-platform build system generator | `-S` source dir, `-B` build dir, `-G` generator, `-D` define variable, `--build`, `--install` |
| `autoconf` | Generate configure scripts | (no major options) |
| `automake` | Generate Makefile.in from Makefile.am | `--add-missing`, `--foreign`, `--gnu` |
| `configure` | Configure source code for compilation | `--prefix`, `--enable-*`, `--disable-*`, `--with-*`, `--without-*` |
| `pkg-config` | Return metadata about installed libraries | `--cflags`, `--libs`, `--modversion`, `--list-all` |
| `ld` | GNU linker | `-o` output, `-l` library, `-L` lib path, `-shared`, `-static`, `-r` relocatable |
| `ldd` | Print shared library dependencies | `-v` verbose, `-u` unused |
| `nm` | List symbols from object files | `-D` dynamic, `-g` external, `-u` undefined, `-C` demangle |
| `objdump` | Display information from object files | `-d` disassemble, `-t` symbols, `-h` headers, `-x` all headers, `-s` full contents |
| `readelf` | Display info about ELF files | `-h` header, `-S` sections, `-l` program headers, `-s` symbols, `-d` dynamic, `-a` all |
| `strip` | Discard symbols from object files | `--strip-all`, `--strip-debug`, `--strip-unneeded` |
| `ar` | Create/modify static libraries | `r` insert, `x` extract, `t` list, `d` delete, `s` index |
| `ranlib` | Generate index to archive | (no major options) |
| `gdb` | GNU Debugger | `-p` attach PID, `-c` core file, `-x` exec commands, `-q` quiet, `-batch` batch mode |
| `valgrind` | Memory debugging and profiling | `--leak-check=full`, `--track-origins=yes`, `--tool=` (memcheck/callgrind/helgrind/massif) |
| `perf` | Performance analysis tool | `stat`, `record`, `report`, `top`, `annotate`, `list`, `-g` call graph |
| `strace` | Trace system calls and signals | `-p` PID, `-f` follow forks, `-e` filter, `-o` output, `-c` summary, `-t` timestamps |
| `ltrace` | Trace library calls | `-p` PID, `-f` forks, `-e` filter, `-o` output, `-c` summary |
| `ctags` | Generate tag file for source code | `-R` recursive, `--languages`, `--exclude`, `-f` output file |
| `cscope` | Interactive source code browser | `-b` build only, `-R` recursive, `-q` fast symbol lookup |
| `indent` | Reformat C source code | `-kr` K&R style, `-gnu` GNU style, `-linux` Linux style |
| `clang-format` | Format C/C++/Java/JS code | `-style`, `-i` in-place, `--dump-config` |
| `python` / `python3` | Python interpreter | `-c` command, `-m` module, `-i` interactive, `-u` unbuffered, `-v` verbose, `-O` optimize |
| `node` | Node.js JavaScript runtime | `-e` evaluate, `-p` print eval, `--inspect` debugger, `--version` |
| `java` | Java application launcher | `-cp` classpath, `-jar` JAR file, `-Xmx` max heap, `-Xms` initial heap, `-D` property |
| `javac` | Java compiler | `-d` output dir, `-cp` classpath, `-source` version, `-target` version |
| `go` | Go programming language tool | `build`, `run`, `test`, `get`, `mod`, `fmt`, `vet`, `install` |
| `rustc` | Rust compiler | `-o` output, `--edition`, `-O` optimize, `--crate-type` |
| `perl` | Perl interpreter | `-e` execute, `-p` line processing, `-n` line loop, `-i` in-place, `-w` warnings |
| `ruby` | Ruby interpreter | `-e` execute, `-p` line processing, `-n` line loop, `-i` in-place, `-w` warnings |
| `php` | PHP interpreter | `-r` execute, `-S` built-in server, `-l` lint, `-i` info, `-m` modules |

---

## 27. SYSTEM ADMINISTRATION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `sudo` | Execute command as superuser | `-u` user, `-i` login shell, `-s` shell, `-l` list, `-k` reset timestamp |
| `su` | Switch user | `-` login shell, `-c` command, `-s` shell |
| `visudo` | Edit sudoers file safely | `-f` file, `-c` check |
| `chroot` | Change root directory | (specify new root and command) |
| `ulimit` | Control user resource limits | `-a` all, `-n` open files, `-u` processes, `-m` memory, `-f` file size, `-c` core dump size, `-v` virtual memory |
| `sysctl` | Configure kernel parameters at runtime | `-a` all, `-w` set, `-p` load from file |
| `ldconfig` | Configure dynamic linker cache | `-v` verbose, `-p` print cache, `-n` process dirs |
| `alternatives` | Manage symbolic links (RHEL) | `--config`, `--install`, `--remove`, `--list` |
| `update-rc.d` | Install/remove SysV init script links | `defaults`, `enable`, `disable`, `remove` |
| `dpkg-reconfigure` | Reconfigure installed packages | `--frontend`, `--priority` |
| `debconf-set-selections` | Set debconf selections | (pipe selections) |
| `timedatectl` | Control system time and date | `status`, `set-time`, `set-timezone`, `set-ntp` |
| `hostnamectl` | Control hostname | `set-hostname`, `status` |
| `localectl` | Control locale settings | `set-locale`, `set-keymap`, `list-locales` |
| `loginctl` | Control systemd login manager | `list-sessions`, `list-users`, `show-session`, `kill-session`, `terminate-user` |
| `machinectl` | Control systemd machine manager | `list`, `status`, `login`, `shell`, `start`, `poweroff` |
| `coredumpctl` | Retrieve stored core dumps | `list`, `info`, `dump`, `gdb`, `-1` latest |
| `busctl` | Introspect D-Bus | `list`, `monitor`, `call`, `introspect`, `tree` |
| `resolvectl` | Resolve DNS names via systemd-resolved | `query`, `status`, `flush-caches`, `dns`, `domain` |
| `networkctl` | Query systemd-networkd | `list`, `status`, `lldp`, `delete` |

---

## 28. CRON & SCHEDULING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `crontab` | Manage user cron jobs | `-e` edit, `-l` list, `-r` remove, `-u` user, `-i` confirm before remove |
| `at` | Schedule one-time future command | `-l` list (atq), `-d` delete (atrm), `-f` file, `-m` mail, `now + 5 minutes` |
| `atq` | List pending at jobs | (no major options) |
| `atrm` | Remove at job | (specify job number) |
| `batch` | Run command when load is low | (no major options) |
| `anacron` | Execute periodic jobs on non-24/7 machines | `-f` force, `-u` update timestamp, `-s` serialize |
| `systemd-run` | Run transient systemd unit | `--on-calendar`, `--on-boot`, `--timer-property`, `--unit`, `--scope`, `--user` |

---

## 29. PERFORMANCE & PROFILING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `top` | Real-time system resource monitor | `-d` delay, `-p` PID, `-u` user, `-b` batch, `-n` iterations |
| `htop` | Interactive process viewer | `-d` delay, `-u` user, `-p` PID, `-t` tree |
| `atop` | Advanced system and process monitor | `-r` read from file, `-w` write to file, interval |
| `glances` | Cross-platform monitoring tool | `-w` web mode, `-s` server, `-c` client, `--export` |
| `nmon` | Performance monitoring tool (AIX/Linux) | (interactive with single-key commands) |
| `iotop` | Monitor disk I/O by process | `-o` only active, `-P` processes, `-a` accumulated, `-d` delay |
| `iftop` | Monitor network bandwidth by connection | `-i` interface, `-n` no DNS, `-N` no port names, `-P` show ports |
| `nethogs` | Monitor network bandwidth per process | `-d` delay, `-v` mode, `-t` tracemode |
| `bmon` | Bandwidth monitor and rate estimator | `-p` interface, `-o` output module |
| `dstat` | Versatile resource statistics tool | `-c` CPU, `-d` disk, `-n` network, `-m` memory, `--top-cpu`, `--top-mem`, `--top-io` |
| `vmstat` | Virtual memory statistics | `interval` `count`, `-s` event counter, `-d` disk, `-w` wide |
| `iostat` | CPU and I/O statistics | `-c` CPU, `-d` disk, `-x` extended, `-h` human |
| `mpstat` | Per-processor statistics | `-P` CPU, `interval`, `count` |
| `sar` | System activity report | `-u` CPU, `-r` memory, `-b` I/O, `-d` disk, `-n` network, `-q` load |
| `perf` | Linux performance counters | `stat`, `record`, `report`, `top`, `annotate` |
| `slabtop` | Display kernel slab cache info in real-time | `-d` delay, `-s` sort, `-o` once |
| `numastat` | NUMA memory statistics | `-m` per-node, `-n` node info, `-p` PID |
| `tload` | Terminal-based load average graph | `-d` delay, `-s` scale |
| `collectl` | Collect and display system performance data | `-s` subsystems, `-p` playback, `-f` file |
| `sysbench` | System performance benchmark | `--test=cpu/memory/fileio/threads`, `--num-threads`, `--max-requests` |
| `stress` | Generate system load for testing | `-c` CPU workers, `-m` memory workers, `-d` disk workers, `-t` timeout |
| `stress-ng` | Advanced stress testing | `--cpu`, `--vm`, `--io`, `--hdd`, `--timeout`, `--metrics` |
| `fio` | Flexible I/O benchmark | `--name`, `--rw`, `--bs`, `--size`, `--numjobs`, `--runtime`, `--ioengine` |

---

## 30. KERNEL & MODULES

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `uname` | Print kernel information | `-r` release, `-a` all, `-m` machine, `-s` name |
| `lsmod` | Show loaded kernel modules | (no major options) |
| `modprobe` | Load/unload kernel modules | `-r` remove, `-v` verbose, `--show-depends`, `-n` dry run |
| `insmod` | Insert kernel module (low-level) | (specify module path) |
| `rmmod` | Remove kernel module (low-level) | `-f` force, `-v` verbose |
| `modinfo` | Show kernel module information | `-p` params, `-d` description, `-a` author |
| `depmod` | Generate module dependency list | `-a` all, `-n` dry run |
| `sysctl` | Configure kernel parameters | `-a` all, `-w` write, `-p` load file |
| `dmesg` | Kernel ring buffer messages | `-T` timestamps, `-w` follow, `-l` level |
| `kmod` | Program to manage kernel modules | `list`, `static-nodes` |
| `dracut` | Generate initramfs | `-f` force, `--kver` kernel version |
| `mkinitramfs` | Create initramfs (Debian) | `-o` output |
| `mkinitcpio` | Create initramfs (Arch) | `-p` preset, `-g` generate |

---

## 31. SIGNALS

| Signal | Number | Description |
|--------|--------|-------------|
| `SIGHUP` | 1 | Hangup — terminal closed or config reload |
| `SIGINT` | 2 | Interrupt — Ctrl+C |
| `SIGQUIT` | 3 | Quit — Ctrl+\\ (with core dump) |
| `SIGKILL` | 9 | Force kill — cannot be caught or ignored |
| `SIGTERM` | 15 | Graceful termination — default kill signal |
| `SIGSTOP` | 19 | Stop process — cannot be caught |
| `SIGCONT` | 18 | Continue stopped process |
| `SIGUSR1` | 10 | User-defined signal 1 |
| `SIGUSR2` | 12 | User-defined signal 2 |
| `SIGCHLD` | 17 | Child process status changed |
| `SIGALRM` | 14 | Timer signal from alarm() |
| `SIGPIPE` | 13 | Broken pipe |
| `SIGSEGV` | 11 | Segmentation fault |

---

## 32. MISCELLANEOUS / UTILITY

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `man` | Display manual pages | `-k` search by keyword (apropos), `-f` short description (whatis), section number (1-8) |
| `info` | Read GNU Info documents | `-f` file, `-n` node |
| `apropos` | Search man page descriptions | `-s` section, `-e` exact match |
| `whatis` | Display one-line man page descriptions | `-s` section |
| `help` | Display help for shell builtins | `-d` short description, `-m` man-like format |
| `type` | Show how command is interpreted | `-a` all, `-t` type only |
| `which` | Show full path of command | `-a` all matches |
| `whereis` | Locate binary, source, and man page | `-b` binaries, `-m` manuals, `-s` sources |
| `cal` | Display a calendar | `-3` three months, `-y` full year |
| `date` | Display or set date and time | `-u` UTC, `+FORMAT` format, `-d` parse string, `-s` set |
| `uptime` | Show uptime and load averages | `-p` pretty, `-s` since |
| `yes` | Output a string repeatedly until killed | (specify string) |
| `seq` | Generate number sequence | `-s` separator, `-w` equal width, `-f` format |
| `sleep` | Delay for specified duration | (number with optional suffix: s/m/h/d) |
| `watch` | Run command periodically, show output | `-n` interval, `-d` differences, `-t` no title |
| `xargs` | Build command lines from standard input | `-I{}` replace, `-n` max args, `-P` parallel, `-0` null delimited |
| `tee` | Duplicate stdin to stdout and file | `-a` append |
| `script` | Record terminal session | `-a` append, `-c` command, `-q` quiet, `-t` timing |
| `screen` | Terminal multiplexer | `-S` session name, `-r` reattach, `-d` detach, `-ls` list, `-X` send command |
| `tmux` | Terminal multiplexer (modern) | `new-session`, `attach`, `detach`, `list-sessions`, `split-window`, `new-window`, `-d` detach, `-s` session name |
| `byobu` | Enhanced screen/tmux wrapper | (toggle keybindings with F9) |
| `expect` | Automate interactive command-line programs | `spawn`, `expect`, `send`, `interact`, `timeout` |
| `bc` | Calculator language | `-l` math library, `scale` precision |
| `dc` | Reverse-polish calculator | (`p` print, `q` quit, stack-based) |
| `units` | Unit conversion tool | `-v` verbose, `-f` units file |
| `factor` | Print prime factors | (no major options) |
| `calendar` | Display reminders | `-A` days ahead, `-B` days back |
| `banner` | Print large banner text | (specify text) |
| `figlet` | Create ASCII art text | `-f` font, `-w` width, `-c` center |
| `toilet` | Enhanced ASCII art text generator | `-f` font, `--metal`, `--gay`, `-w` width |
| `cowsay` | Generate ASCII art cow with message | `-f` cow file, `-l` list cows, `-e` eyes, `-T` tongue |
| `fortune` | Display random quotation | `-s` short, `-l` long, `-a` all |
| `lolcat` | Rainbow-colored cat | `-a` animate, `-s` speed, `-f` frequency |
| `tree` | Display directory tree structure | `-L` depth, `-d` dirs only, `-a` all files, `-h` human-readable sizes, `-I` exclude pattern, `--dirsfirst`, `-f` full path, `-p` permissions |
| `mc` | Midnight Commander file manager | `-a` slow terminal, `-b` B&W, `-c` color |
| `ranger` | Console file manager with vim keybindings | `--copy-config` generate config |
| `nnn` | Fast terminal file manager | `-d` detail mode, `-H` hidden files, `-n` type-to-nav |
| `dialog` | Display dialog boxes in shell scripts | `--msgbox`, `--yesno`, `--inputbox`, `--menu`, `--checklist`, `--title` |
| `whiptail` | Display dialog boxes (Newt-based) | `--msgbox`, `--yesno`, `--inputbox`, `--menu`, `--title`, `--backtitle` |
| `zenity` | Display GTK dialog boxes | `--info`, `--warning`, `--error`, `--question`, `--entry`, `--file-selection`, `--list`, `--progress` |
| `notify-send` | Send desktop notifications | `-u` urgency, `-t` timeout, `-i` icon, `-a` app name |
| `xdg-open` | Open file/URL with default application | (no major options) |
| `open` | Open file with associated program | (macOS; aliased on some Linux) |
| `xclip` | Command-line clipboard access (X11) | `-selection clipboard`, `-i` input, `-o` output |
| `xsel` | Command-line clipboard access (X11) | `--clipboard`, `--input`, `--output`, `--clear` |
| `wl-copy` / `wl-paste` | Wayland clipboard tool | `-n` no newline, `--type` MIME type |
| `parallel` | Execute commands in parallel | `-j` jobs, `--eta` ETA, `--progress`, `--halt`, `--dry-run`, `-k` keep order, `--pipe` |
| `cmp` | Byte-by-byte file comparison | `-b` print bytes, `-l` all diffs, `-s` silent |
| `md5sum` | Compute MD5 checksum | `-c` check |
| `sha256sum` | Compute SHA-256 checksum | `-c` check |
| `sync` | Flush filesystem buffers | (no major options) |
| `tasksel` | Install predefined software groups (Debian) | `--list-tasks`, `--task-packages` |

---

## 33. FILESYSTEM TYPES & MOUNTING

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `mount` | Mount filesystem | `-t` type, `-o` options, `-a` all fstab, `--bind`, `-r` readonly |
| `umount` | Unmount filesystem | `-l` lazy, `-f` force |
| `findmnt` | Find mounted filesystems | `-t` type, `-n` no headers, `-r` raw, `-l` list, `-D` df-like |
| `mountpoint` | Check if directory is a mountpoint | `-q` quiet |
| `eject` | Eject removable media | `-t` close tray, `-r` CDROM |
| `udisksctl` | udisks2 CLI for managing disks | `mount`, `unmount`, `power-off`, `info`, `status`, `loop-setup` |

---

## 34. LOCALE & INTERNATIONALIZATION

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `locale` | Show current locale settings | `-a` all locales, `-m` charmaps |
| `localectl` | Control system locale (systemd) | `status`, `set-locale`, `list-locales`, `set-keymap` |
| `locale-gen` | Generate locale files | (edit `/etc/locale.gen` first) |
| `iconv` | Convert text character encoding | `-f` from, `-t` to, `-l` list, `-o` output |
| `recode` | Charset converter | (charset1..charset2) |

---

## 35. SYSTEM RESCUE & RECOVERY

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `fsck` | Check and repair filesystem | `-t` type, `-a` auto, `-y` yes, `-n` no, `-f` force |
| `e2fsck` | Check ext2/3/4 filesystem | `-p` auto, `-y` yes, `-f` force |
| `xfs_repair` | Repair XFS filesystem | `-n` no modify, `-v` verbose |
| `btrfs check` | Check Btrfs filesystem | `--repair` (dangerous), `--readonly` |
| `testdisk` | Recover lost partitions | (interactive menus) |
| `photorec` | Recover deleted files by file type | (interactive menus) |
| `ddrescue` | Data recovery tool | `-r` retries, `-n` no scrape, `-d` direct access, `-f` force |
| `foremost` | Recover files by headers/footers | `-t` types, `-o` output dir, `-i` input |
| `scalpel` | File carving and recovery | `-o` output, `-c` config |
| `extundelete` | Recover deleted files from ext3/ext4 | `--restore-file`, `--restore-all`, `--after` |

---

## 36. POWER MANAGEMENT

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `shutdown` | Halt, power off, or reboot | `-h` halt, `-r` reboot, `-c` cancel, `now`, `+minutes` |
| `reboot` | Restart system | `-f` force |
| `poweroff` | Power off system | `-f` force |
| `halt` | Halt system | `-f` force, `-p` poweroff |
| `suspend` | Suspend system (via systemctl) | `systemctl suspend` |
| `hibernate` | Hibernate system (via systemctl) | `systemctl hibernate` |
| `pm-suspend` | Suspend to RAM | (no major options) |
| `pm-hibernate` | Suspend to disk | (no major options) |
| `acpi` | Show battery, thermal, and AC info | `-b` battery, `-t` thermal, `-a` AC adapter, `-V` verbose |
| `tlp` | Advanced power management for laptops | `start`, `bat`, `ac`, `stat`, `--version` |
| `powertop` | Power consumption analyzer | `--auto-tune`, `--html`, `--csv`, `--time` |
| `cpupower` | CPU frequency scaling | `frequency-info`, `frequency-set`, `idle-info`, `-g` governor |
| `cpufreq-info` | Show CPU frequency info | `-p` policy, `-f` freq, `-w` hardware limits |
| `cpufreq-set` | Set CPU frequency | `-g` governor, `-f` frequency, `-d` min, `-u` max |
| `turbostat` | Report CPU topology and power | `-i` interval, `-c` CPU, `-S` summary |

---

## 37. AUDIO & VIDEO

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `alsamixer` | ALSA sound mixer (TUI) | `-c` card number, `-V` view mode |
| `amixer` | ALSA sound mixer (CLI) | `sset` set control, `sget` get control, `-c` card |
| `aplay` | Play audio files (ALSA) | `-l` list devices, `-D` device, `-f` format, `-r` rate |
| `arecord` | Record audio (ALSA) | `-l` list devices, `-D` device, `-f` format, `-r` rate, `-d` duration |
| `pactl` | PulseAudio control | `list`, `set-sink-volume`, `set-source-volume`, `set-default-sink`, `info` |
| `pavucontrol` | PulseAudio volume control (GUI) | (no major options) |
| `pulseaudio` | PulseAudio sound server | `--start`, `--kill`, `--check`, `-D` daemon |
| `pipewire` | Modern audio/video server | (runs as service) |
| `wpctl` | WirePlumber (PipeWire) control | `status`, `set-volume`, `set-mute`, `inspect` |
| `pw-cli` | PipeWire command-line interface | `list-objects`, `info`, `dump` |
| `ffmpeg` | Universal audio/video converter | `-i` input, `-c:v` video codec, `-c:a` audio codec, `-b:v` video bitrate, `-b:a` audio bitrate, `-r` framerate, `-s` resolution, `-ss` start, `-t` duration, `-f` format, `-y` overwrite |
| `ffprobe` | Analyze multimedia file info | `-show_format`, `-show_streams`, `-v quiet`, `-print_format json` |
| `sox` | Sound processing Swiss Army knife | `--combine mix`, `rate`, `channels`, `trim`, `fade`, `gain`, `speed`, `reverse` |
| `mpv` | Media player (CLI) | `--no-video`, `--loop`, `--volume`, `--speed`, `--fullscreen` |
| `vlc` | VLC media player | `--intf ncurses`, `--play-and-exit`, `--no-video`, `--rate` |
| `mplayer` | Media player | `-vo null` no video, `-ao` audio output, `-ss` seek, `-endpos` duration |
| `convert` (ImageMagick) | Convert/transform images | `-resize`, `-rotate`, `-quality`, `-format`, `-crop`, `-flip`, `-flop`, `-monochrome` |
| `identify` (ImageMagick) | Describe image format and characteristics | `-verbose`, `-format` |
| `mogrify` (ImageMagick) | In-place image transformation | `-resize`, `-format`, `-quality`, `-path` output dir |
| `composite` (ImageMagick) | Overlay images | `-gravity`, `-dissolve`, `-blend` |
| `display` (ImageMagick) | Display image on X server | (no major CLI options) |
| `gimp` | GNU Image Manipulation Program | `-i` no UI, `-b` batch command, `-d` no data, `-f` no fonts |
| `inkscape` | Vector graphics editor | `--export-png`, `--export-pdf`, `--export-svg`, `-w` width, `-h` height |

---

## 38. NETWORKING DIAGNOSTICS

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `ping` | Test network connectivity | `-c` count, `-i` interval, `-s` size, `-W` timeout |
| `traceroute` | Trace packet route to destination | `-n` numeric, `-m` max hops, `-I` ICMP |
| `mtr` | Real-time traceroute | `-r` report, `-c` count, `-n` no DNS |
| `dig` | DNS lookup utility | `+short`, `+trace`, `-t` type, `@server` |
| `nslookup` | DNS query tool | `-type=` record type |
| `host` | Simple DNS lookup | `-t` type, `-a` all |
| `ss` | Socket statistics | `-t` TCP, `-u` UDP, `-l` listening, `-p` process, `-n` numeric |
| `netstat` | Network statistics (legacy) | `-tulnp` common combo |
| `tcpdump` | Packet capture tool | `-i` interface, `-n` no DNS, `-w` write file, `-r` read file |
| `nmap` | Network scanner | `-sS` SYN scan, `-p` ports, `-O` OS detect, `-sV` version |
| `telnet` | Interactive TCP connection | (host port) |
| `nc` | Netcat — read/write TCP/UDP connections | `-l` listen, `-p` port, `-z` scan, `-v` verbose |
| `iperf3` | Network throughput measurement | `-s` server, `-c` client, `-u` UDP, `-t` time |
| `ethtool` | Network interface diagnostics | `-i` driver, `-S` stats, `-s` change settings |
| `nmcli` | NetworkManager CLI | `device`, `connection`, `general`, `radio` |
| `ip` | Network interface and routing management | `addr`, `link`, `route`, `neigh` |
| `arp` | ARP table management (legacy) | `-a` display, `-d` delete |
| `resolvectl` | DNS resolution via systemd | `query`, `status`, `flush-caches` |
| `whois` | Domain/IP WHOIS lookup | `-h` server |
| `curl` | Transfer data via URL protocols | `-I` headers only, `-v` verbose, `-s` silent |
| `wget` | Download files | `-c` continue, `-r` recursive, `-O` output |

---

## 39. HARDWARE & DEVICE MANAGEMENT

| Command | Description | Important Options |
|---------|-------------|-------------------|
| `lspci` | List PCI devices | `-v` verbose, `-k` kernel drivers, `-nn` IDs, `-t` tree |
| `lsusb` | List USB devices | `-v` verbose, `-t` tree |
| `lsblk` | List block devices | `-f` filesystems, `-o` columns, `-a` all |
| `lscpu` | Display CPU info | `-e` extended, `-p` parseable |
| `lshw` | List hardware info | `-short`, `-class`, `-html` |
| `lsscsi` | List SCSI devices | `-g` generic, `-s` size |
| `dmidecode` | DMI/BIOS hardware info | `-t` type (bios/memory/processor) |
| `hdparm` | Disk parameter tool | `-i` info, `-t` read timing, `-T` cache timing |
| `smartctl` | SMART disk monitoring | `-a` all, `-H` health, `-t` test |
| `udevadm` | udev device manager tool | `info`, `trigger`, `settle`, `monitor`, `test`, `control` |
| `lsdev` | Display hardware device info | (procinfo package) |
| `hwinfo` | Hardware information tool (SUSE) | `--short`, `--cpu`, `--disk`, `--network` |
| `sensors` | Hardware sensor monitoring (lm-sensors) | `-f` Fahrenheit, `-u` raw values |
| `sensors-detect` | Detect hardware sensors | (interactive) |
| `acpi` | Battery/thermal info | `-b` battery, `-t` thermal |
| `inxi` | Full system information script | `-F` full, `-G` graphics, `-A` audio, `-N` network, `-D` disks |

---

## 40. SPECIAL FILES & DIRECTORIES

| Path | Description |
|------|-------------|
| `/etc/passwd` | User account information |
| `/etc/shadow` | Encrypted passwords |
| `/etc/group` | Group information |
| `/etc/sudoers` | Sudo configuration |
| `/etc/fstab` | Filesystem mount table |
| `/etc/hosts` | Static hostname-to-IP mappings |
| `/etc/hostname` | System hostname |
| `/etc/resolv.conf` | DNS resolver configuration |
| `/etc/network/interfaces` | Network config (Debian) |
| `/etc/sysconfig/network-scripts/` | Network config (RHEL) |
| `/etc/ssh/sshd_config` | SSH server configuration |
| `/etc/crontab` | System cron jobs |
| `/etc/profile` | System-wide shell profile |
| `/etc/bashrc` or `/etc/bash.bashrc` | System-wide bash config |
| `~/.bashrc` | User bash interactive config |
| `~/.bash_profile` | User bash login config |
| `~/.ssh/` | User SSH keys and config |
| `/var/log/` | System log files |
| `/var/log/syslog` or `/var/log/messages` | Main system log |
| `/var/log/auth.log` or `/var/log/secure` | Authentication log |
| `/proc/` | Virtual filesystem — process and kernel info |
| `/sys/` | Virtual filesystem — device and driver info |
| `/dev/` | Device files |
| `/dev/null` | Discard output (black hole) |
| `/dev/zero` | Infinite zero bytes |
| `/dev/urandom` | Random data source |
| `/tmp/` | Temporary files (cleared on reboot) |

---

*Total commands covered: 600+*
*Last updated: April 2026*
