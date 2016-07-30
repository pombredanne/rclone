---
title: "Filtering"
description: "Filtering, includes and excludes"
date: "2016-02-09"
---

# Filtering, includes and excludes #

Rclone has a sophisticated set of include and exclude rules. Some of
these are based on patterns and some on other things like file size.

The filters are applied for the `copy`, `sync`, `move`, `ls`, `lsl`,
`md5sum`, `sha1sum`, `size`, `delete` and `check` operations.
Note that `purge` does not obey the filters.

Each path as it passes through rclone is matched against the include
and exclude rules like `--include`, `--exclude`, `--include-from`,
`--exclude-from`, `--filter`, or `--filter-from`. The simplest way to
try them out is using the `ls` command, or `--dry-run` together with
`-v`.

**Important** Due to limitations of the command line parser you can
only use any of these options once - if you duplicate them then rclone
will use the last one only.

## Patterns ##

The patterns used to match files for inclusion or exclusion are based
on "file globs" as used by the unix shell.

If the pattern starts with a `/` then it only matches at the top level
of the directory tree, relative to the root of the remote.
If it doesn't start with `/` then it is matched starting at the
**end of the path**, but it will only match a complete path element:

    file.jpg  - matches "file.jpg"
              - matches "directory/file.jpg"
              - doesn't match "afile.jpg"
              - doesn't match "directory/afile.jpg"
    /file.jpg - matches "file.jpg" in the root directory of the remote
              - doesn't match "afile.jpg"
              - doesn't match "directory/file.jpg"

**Important** Note that you must use `/` in patterns and not `\` even
if running on Windows.

A `*` matches anything but not a `/`.

    *.jpg  - matches "file.jpg"
           - matches "directory/file.jpg"
           - doesn't match "file.jpg/something"

Use `**` to match anything, including slashes (`/`).

    dir/** - matches "dir/file.jpg"
           - matches "dir/dir1/dir2/file.jpg"
           - doesn't match "directory/file.jpg"
           - doesn't match "adir/file.jpg"

A `?` matches any character except a slash `/`.

    l?ss  - matches "less"
          - matches "lass"
          - doesn't match "floss"

A `[` and `]` together make a a character class, such as `[a-z]` or
`[aeiou]` or `[[:alpha:]]`.  See the [go regexp
docs](https://golang.org/pkg/regexp/syntax/) for more info on these.

    h[ae]llo - matches "hello"
             - matches "hallo"
             - doesn't match "hullo"

A `{` and `}` define a choice between elements.  It should contain a
comma seperated list of patterns, any of which might match.  These
patterns can contain wildcards.

    {one,two}_potato - matches "one_potato"
                     - matches "two_potato"
                     - doesn't match "three_potato"
                     - doesn't match "_potato"

Special characters can be escaped with a `\` before them.

    \*.jpg       - matches "*.jpg"
    \\.jpg       - matches "\.jpg"
    \[one\].jpg  - matches "[one].jpg"

Note also that rclone filter globs can only be used in one of the
filter command line flags, not in the specification of the remote, so
`rclone copy "remote:dir*.jpg" /path/to/dir` won't work - what is
required is `rclone --include "*.jpg" copy remote:dir /path/to/dir`

### Directories ###

Rclone keeps track of directories that could match any file patterns.

Eg if you add the include rule

    \a\*.jpg

Rclone will synthesize the directory include rule

    \a\

If you put any rules which end in `\` then it will only match
directories.

Directory matches are **only** used to optimise directory access
patterns - you must still match the files that you want to match.
Directory matches won't optimise anything on bucket based remotes (eg
s3, swift, google compute storage, b2) which don't have a concept of
directory.

### Differences between rsync and rclone patterns ###

Rclone implements bash style `{a,b,c}` glob matching which rsync doesn't.

Rclone always does a wildcard match so `\` must always escape a `\`.

## How the rules are used ##

Rclone maintains a list of include rules and exclude rules.

Each file is matched in order against the list until it finds a match.
The file is then included or excluded according to the rule type.

If the matcher falls off the bottom of the list then the path is
included.

For example given the following rules, `+` being include, `-` being
exclude,

    - secret*.jpg
    + *.jpg
    + *.png
    + file2.avi
    - *

This would include

  * `file1.jpg`
  * `file3.png`
  * `file2.avi`

This would exclude

  * `secret17.jpg`
  * non `*.jpg` and `*.png`

A similar process is done on directory entries before recursing into
them.  This only works on remotes which have a concept of directory
(Eg local, google drive, onedrive, amazon drive) and not on bucket
based remotes (eg s3, swift, google compute storage, b2).

## Adding filtering rules ##

Filtering rules are added with the following command line flags.

### `--exclude` - Exclude files matching pattern ###

Add a single exclude rule with `--exclude`.

Eg `--exclude *.bak` to exclude all bak files from the sync.

### `--exclude-from` - Read exclude patterns from file ###

Add exclude rules from a file.

Prepare a file like this `exclude-file.txt`

    # a sample exclude rule file
    *.bak
    file2.jpg

Then use as `--exclude-from exclude-file.txt`.  This will sync all
files except those ending in `bak` and `file2.jpg`.

This is useful if you have a lot of rules.

### `--include` - Include files matching pattern ###

Add a single include rule with `--include`.

Eg `--include *.{png,jpg}` to include all `png` and `jpg` files in the
backup and no others.

This adds an implicit `--exclude *` at the very end of the filter
list. This means you can mix `--include` and `--include-from` with the
other filters (eg `--exclude`) but you must include all the files you
want in the include statement.  If this doesn't provide enough
flexibility then you must use `--filter-from`.

### `--include-from` - Read include patterns from file ###

Add include rules from a file.

Prepare a file like this `include-file.txt`

    # a sample include rule file
    *.jpg
    *.png
    file2.avi

Then use as `--include-from include-file.txt`.  This will sync all
`jpg`, `png` files and `file2.avi`.

This is useful if you have a lot of rules.

This adds an implicit `--exclude *` at the very end of the filter
list. This means you can mix `--include` and `--include-from` with the
other filters (eg `--exclude`) but you must include all the files you
want in the include statement.  If this doesn't provide enough
flexibility then you must use `--filter-from`.

### `--filter` - Add a file-filtering rule ###

This can be used to add a single include or exclude rule.  Include
rules start with `+ ` and exclude rules start with `- `.  A special
rule called `!` can be used to clear the existing rules.

Eg `--filter "- *.bak"` to exclude all bak files from the sync.

### `--filter-from` - Read filtering patterns from a file ###

Add include/exclude rules from a file.

Prepare a file like this `filter-file.txt`

    # a sample exclude rule file
    - secret*.jpg
    + *.jpg
    + *.png
    + file2.avi
    # exclude everything else
    - *

Then use as `--filter-from filter-file.txt`.  The rules are processed
in the order that they are defined.

This example will include all `jpg` and `png` files, exclude any files
matching `secret*.jpg` and include `file2.avi`.  Everything else will
be excluded from the sync.

### `--files-from` - Read list of source-file names ###

This reads a list of file names from the file passed in and **only**
these files are transferred.  The filtering rules are ignored
completely if you use this option.

Prepare a file like this `files-from.txt`

    # comment
    file1.jpg
    file2.jpg

Then use as `--files-from files-from.txt`.  This will only transfer
`file1.jpg` and `file2.jpg` providing they exist.

For example, let's say you had a few files you want to back up
regularly with these absolute paths:

    /home/user1/important
    /home/user1/dir/file
    /home/user2/stuff

To copy these you'd find a common subdirectory - in this case `/home`
and put the remaining files in `files-from.txt` with or without
leading `/`, eg

    user1/important
    user1/dir/file
    user2/stuff

You could then copy these to a remote like this

    rclone copy --files-from files-from.txt /home remote:backup

The 3 files will arrive in `remote:backup` with the paths as in the
`files-from.txt`.

You could of course choose `/` as the root too in which case your
`files-from.txt` might look like this.

    /home/user1/important
    /home/user1/dir/file
    /home/user2/stuff

And you would transfer it like this

    rclone copy --files-from files-from.txt / remote:backup

In this case there will be an extra `home` directory on the remote.

### `--min-size` - Don't transfer any file smaller than this ###

This option controls the minimum size file which will be transferred.
This defaults to `kBytes` but a suffix of `k`, `M`, or `G` can be
used.

For example `--min-size 50k` means no files smaller than 50kByte will be
transferred.

### `--max-size` - Don't transfer any file larger than this ###

This option controls the maximum size file which will be transferred.
This defaults to `kBytes` but a suffix of `k`, `M`, or `G` can be
used.

For example `--max-size 1G` means no files larger than 1GByte will be
transferred.

### `--max-age` - Don't transfer any file older than this ###

This option controls the maximum age of files to transfer.  Give in
seconds or with a suffix of:

  * `ms` - Milliseconds
  * `s` - Seconds
  * `m` - Minutes
  * `h` - Hours
  * `d` - Days
  * `w` - Weeks
  * `M` - Months
  * `y` - Years

For example `--max-age 2d` means no files older than 2 days will be
transferred.

### `--min-age` - Don't transfer any file younger than this ###

This option controls the minimum age of files to transfer.  Give in
seconds or with a suffix (see `--max-age` for list of suffixes)

For example `--min-age 2d` means no files younger than 2 days will be
transferred.

### `--delete-excluded` - Delete files on dest excluded from sync ###

**Important** this flag is dangerous - use with `--dry-run` and `-v` first.

When doing `rclone sync` this will delete any files which are excluded
from the sync on the destination.

If for example you did a sync from `A` to `B` without the `--min-size 50k` flag

    rclone sync A: B:

Then you repeated it like this with the `--delete-excluded`

    rclone --min-size 50k --delete-excluded sync A: B:

This would delete all files on `B` which are less than 50 kBytes as
these are now excluded from the sync.

Always test first with `--dry-run` and `-v` before using this flag.

### `--dump-filters` - dump the filters to the output ###

This dumps the defined filters to the output as regular expressions.

Useful for debugging.

## Quoting shell metacharacters ##

The examples above may not work verbatim in your shell as they have
shell metacharacters in them (eg `*`), and may require quoting.

Eg linux, OSX

  * `--include \*.jpg`
  * `--include '*.jpg'`
  * `--include='*.jpg'`

In Windows the expansion is done by the command not the shell so this
should work fine

  * `--include *.jpg`
