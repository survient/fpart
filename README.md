        _______ ____   __         _      __
       / /  ___|  _ \ / /_ _ _ __| |_   / /
      / /| |_  | |_) / / _` | '__| __| / /
     / / |  _| |  __/ / (_| | |  | |_ / /
    /_/  |_|   |_| /_/ \__,_|_|   \__/_/

What is fpart ?
===============

Fpart is a tool that helps you sort file trees and pack them into bags (called
"partitions"). It is developed in C and available under the BSD license.

It splits a list of directories and file trees into a certain number of
partitions, trying to produce partitions with the same size and number of files.
It can also produce partitions with a given number of files or of a limited
size. Fpart uses a bin packing algorithm to optimize space utilization amongst
partitions.

Once generated, partitions are either printed as file lists to stdout (default)
or to files. Those lists can then be used by third party programs.

Fpart also includes a live mode, which allows it to crawl very large filesystems
and produce partitions in live. Hooks are available to act on those partitions
(e.g. immediately start a transfer using rsync(1) or cpio(1)) without having to
wait for the filesystem traversal job to be finished. Used that way, fpart can
be seen as a powerful basis for a data migration tool.

Fpart can also generate lists of directories instead of files. That mode can
be useful to enable usage of options requiring overall knowledge of directories
such as rsync's --delete.

As a demonstration of fpart possibilities, a tool called fpsync is provided in
the tools/ directory (see also below for more details).

Compatibility :
===============

Fpart is primarily developed on FreeBSD.

It has been successfully tested on :

* FreeBSD (i386, amd64)
* GNU/Linux (x86_64, arm)
* Solaris 9, 10 (Sparc, i386)
* OpenIndiana (i386)
* NetBSD (amd64, alpha)
* Mac OS X (10.6, 10.8)

and will probably work on other operating systems too (*).

(*) fpart built as a static binary within a Debian (armel) chroot will give you
a powerful tool for backing-up your Android (arm) phone ;-)

Examples :
==========

Common usage :
--------------

The following will produce 3 partitions, with (approximatively) the same size
and number of files. Three files: "var-parts.[0-2]", are generated as output :

    $ fpart -n 3 -o var-parts /var
    
    $ ls var-parts*
    var-parts.0 var-parts.1 var-parts.2
    
    $ head -n 2 var-parts.0
    /var/some/file1
    /var/some/file2

The following will produce partitions of 4.3 GB, containing music files ready
to be burnt to a DVD (for example). Files "music-parts.[0-n]", are generated
as output :

    $ fpart -s 4617089843 -o music-parts /path/to/my/music

The following will produce partitions containing 10000 files each by examining
/usr first and then /home and display only partition 0 on stdout :

    $ find /usr ! -type d | fpart -f 10000 -i - /home | grep '^0:'

The following will produce two partitions by re-using du(1) output. Fpart will
not examine the filesystem but instead re-use arbitrary values printed by du(1)
and sort them :

    $ du * | fpart -n 2 -a

Live mode :
-----------

By default, fpart will wait for FS crawling to terminate before generating and
displaying partitions. If you use the live mode (option -L), fpart will display
each partition as soon as it is complete. You can combine that option with
hooks; they will be triggered just before (pre-part hook, option -w) or after
(post-part hook, option -W) partitions' completion.

Hooks provide several environment variables (see fpart(1)); they are a
convenient way of getting information about fpart's and partition's current
states. For example, ${FPART_PARTFILENAME} will contain the name of the output
file of the partition that has just been generated; using that variable within a
post-part hook permits starting manipulating the files just after partition
generation.

See the following example :

    $ mkdir foo && touch foo/{bar,baz}
    $ fpart -L -f 1 -o /tmp/part.out -W \
        'echo == ${FPART_PARTFILENAME} == ; cat ${FPART_PARTFILENAME}' foo/
    == /tmp/part.out.0 ==
    foo/bar
    == /tmp/part.out.1 ==
    foo/baz

This example crawls foo/ in live mode (option -L). For each file (option -f,
1 file per partition), it generates a partition into /tmp/part.out.<n>
(option -o; <n> is the partition index and will be automatically added by fpart)
and executes the following post-part hook (option -W) :

    echo == ${FPART_PARTFILENAME} == ; cat ${FPART_PARTFILENAME}

This hook will display the name of current partition's output file name as well
as display its contents.

Migrating data :
----------------

Here is a more complex example that will show you how to use fpart, GNU Parallel
and Rsync to split up a directory and immediately schedule data synchronization
of smaller lists of files, while FS crawling goes on. We will be synchronizing
data from /data/src to /data/dest.

First, go to the source directory (as rsync's --files-from option takes a file
list relative to its source directory) :

    $ cd /data/src

Then, run fpart from here :

    $ fpart -L -f 10000 -x '.snapshot' -x '.zfs' -zz -o /tmp/part.out -W \
      '/usr/local/bin/sem -j 3
        "/usr/local/bin/rsync -av --files-from=${FPART_PARTFILENAME}
          /data/src/ /data/dest/"' .

This command will start fpart in live mode (option -L), making it generate
partitions during FS crawling. Fpart will produce partitions containing at most
10000 files each (option -f), will skip files and folders named '.snapshot' or
'.zfs' (option -x) and will list empty and non-accessible directories (option
-zz; that option is necessary when working with rsync to make sure the whole file
tree will be re-created within the destination directory). Last but not least,
each partition will be written to /tmp/part.out.<n> (option -o) and used within
the post-part hook (option -W), run immediately by fpart once the partition is
complete :

    /usr/local/bin/sem -j 3
        "/usr/local/bin/rsync -av --files-from=${FPART_PARTFILENAME} /data/src/ /data/dest/"

This hook is itself a nested command. It will run GNU Parallel's sem scheduler
(any other scheduler would do) to run at most 3 rsync jobs in parallel.

The scheduler will finally trigger the following command :

    /usr/local/bin/rsync -av --files-from=${FPART_PARTFILENAME} /data/src/ /data/dest/

where ${FPART_PARTFILENAME} will be part of rsync's environment when it runs
and contains the file name of the partition that has just been generated.

That's all, folks ! Pretty simple, isn't it ?

In this example, FS crawling and data transfer are run from the same -local-
machine, but you can use it as the basis of a much sophisticated solution: at
$work, by using a cluster of machines connected to our filers through NFS and
running Open Grid Scheduler, we successully migrated over 400 TB of data.

Note: several successive fpart runs can be launched using the above example;
you will perform incremental synchronizations. That is, deleted files from the
source directory will not be removed from destination unless rsync's --delete
option is used. Unfortunately, this option cannot be used with a list of files
(files that do not appear in the list are just ignored). To use the --delete
option in conjunction with fpart, you *have* to provide rsync's --files-from
option a list of directories (only); that can be performed using fpart's -E
option.

Fpsync :
========

To demonstrate fpart possibilities, a program called 'fpsync' is provided within
the tools/ directory. This tool is a shell script that wraps fpart(1) and
rsync(1) (or cpio(1)) to launch several synchronization jobs in parallel as
presented in the previous section, but while the previous example used GNU
Parallel to schedule transfers, fpsync provides its own -embedded- scheduler.
It can execute several synchronization processes locally or launch them on
several nodes (workers) through SSH.

Despite its initial "proof of concept" status, fpsync has quickly evolved into
a powerful (yet simple to use) migration tool and has been successfully used
to boost migration of several hundreds of TB of data (initially at $work but it
has also been tested by several organizations such as UCI, Intel and Amazon ;
see the 'See also' section at the end of this document).

In addition to being very fast (as transfers start during FS crawling and are
parallelized), fpsync is able to resume synchronization jobs (see option -r)
and presents an overall progress status. It also has a small memory footprint
compared to rsync itself when migrating filesystems with a big number of files.

Last but not least, fpsync is very easy to set up and only requires a few
(common) software to run: fpart, rsync and/or cpio, a POSIX shell, sudo and ssh.

See fpsync(1) to learn more about that tool and get a list of all supported
options.

Here is a simple representation of how it works :
-------------------------------------------------

    fpsync [args] /data/src/ /data/dst/
      |
      +-- fpart (live mode) crawls /data/src/, generates parts.[1] + sync jobs ->
      |    \    \    \
      |     \    \    +___ part. #n + job #n
      |      \    \
      |       \    +______ part. #1 + job #1
      |        \
      |         +_________ part. #0 + job #0
      |
      +-- fpsync scheduler, executes jobs either locally or remotely ----------->
           \    \    \
            \    \    +___ sync job #n... --------------------------------------> +
             \    \                                                               |
              \    +______ sync job #1 ---------------------------------->        |
               \                                                                  |
                +_________ sync job #0 ----------------------------->             +
                                                                                 /
                                                                                /
                  Filesystem tree rebuilt and synchronized! <------------------+
    
    [1] Either containing file lists (default mode) or directory lists (option -E)

File mode :
-----------

In its default mode, fpsync uses rsync(1) and works with file lists to perform
incremental (only) synchronizations. Using the cpio(1) tool (option -m) will
perform the same kind of synchronizations but using the cpio(1) tool (see
'Notes about cpio tool' below).

The following examples show two typical usage.

The command :

    $ fpsync -n 4 -f 1000 -s $((100 * 1024 * 1024)) \
        /data/src/ /data/dst/

will synchronize /data/src/ to /data/dst/ using 4 local workers, each one
transferring at most 1000 files and 100 MB per synchronization job.

The command :

    $ fpsync -n 8 -f 1000 -s $((100 * 1024 * 1024)) \
        -w login@machine1 -w login@machine2 -d /mnt/nfs/fpsync \
        /data/src/ /data/dst/

will synchronize /data/src/ to /data/dst/ using the same transfer limits, but
through 8 concurrent synchronization jobs spread over two machines (machine1 and
machine2). Those machines must both be able to access /data/src/ and /data/dst/,
as well as /mnt/nfs/fpsync, which is fpsync's shared working directory.

As previously mentioned, those two examples work with file lists and will
perform *incremental* synchronizations. As a consequence, they will require a
final -manual- 'rsync --delete' pass to delete extra files from the /data/dst/
directory.

Directory mode :
----------------

If you want to avoid that final pass, use fpsync's option -E (only compatible
with rsync tool). That option will make fpsync work with a list of *directories*
(instead of files) and will (forcibly) enable rsync's --delete option with each
synchronization job. The counterpart of using that mode is that directory lists
are coarse-grained and will probably be less balanced than file lists. The best
option is probably to run several incremental jobs and keep the -E option to
speed up the final pass only.

(you can read the file 'Solving_the_final_pass_challenge.txt' in the docs/
directory for more details about fpsync's option -E)

Notes about cpio tool :
-----------------------

Fpsync's option '-m' allows you to use cpio(1) instead of rsync(1) to copy
files. Cpio(1) is much faster than rsync(1) but there is a catch: when
re-creating a complex file tree, missing parent directories are created
on-the-fly. In that case, original directory metadata (e.g. timestamps) are
*not* copied from source.

To overcome that limitation, fpsync uses fpart's -zzz option to ask fpart to
also pack every single directory (0-sized) with file lists. Making directories
appear in file lists will ask cpio to copy their metadata when the directory is
processed (of course, fpart ensures that a parent directory entry appears after
files beneath. If the parent directory is missing it is first created on the
fly, then the directory entry makes cpio update its metadata).

This works fine with a single cpio process (fpsync's option -n 1) but not with 2
or more parallel processes which can treat partitions out-of-order. Indeed, if
several workers copy files to the same directory at the same time, it is
possible that the parent directory's original metadata gets re-applied while
another worker is still adding files to that directory. That can occur if a
directory list spreads over more than one partition. In such a situation,
original metadata (here, mtime) gets overwritten while new files get added to
the directory.

That race condition is un-avoidable (fpart would have to guarantee the
directory entry belongs to the *same* partition as its files beneath, that
would probably lead to un-balanced partitions as well as increased -and useless-
complexity).

You've been warned. Anyway, maybe you do not care about copying original
directory mtimes. If this is the case, you can ignore that situation. If you
care about them, running a second pass of fpsync will fix the timestamps.

Notes about GNU cpio (specifically) :
-------------------------------------

Developments have been made with BSD cpio (FreeBSD version). Fpsync will work
with GNU cpio too but there are small behaviour differences you must be aware
of :

- for an unknown reason, GNU cpio will not apply mtime to the main target
directory (AKA './' when received by cpio).

- when using GNU cpio, you will get the following warnings when performing a
second pass :

    <file> not created: newer or same age version exists

You can ignore those warnings as that second pass will fix directory timestamps
anyway.

Warning: if you pass option '-u' to cpio (trough fpsync's option '-o') to get
rid of those messages, you will possibly re-touch directory mtimes (loosing
original ones). Also, be aware of what that option implies: re-transferring
every single file.

Notes about hard links :
------------------------

Rsync can detect and replicate hard links with option -H but that will NOT work
with fpsync because rsync collects hard links' information on a per-run basis.

So, as for directory metadata (see above), being able to propagate hard links
with fpsync would require from fpart the guarantee that all related links belong
to the same partition.

Unfortunately, this is not something fpart can do because, in live mode (used by
fpsync to start synchronization as soon as possible), it crawls the filesystem
as it comes. As a consequence, there is no mean to know if a hard link connected
to a file already written to a partition (and probably already synchronized
through an independent rsync process) will appear later or not. Also, in
non-live mode, trying to group related hardlinks into the same partitions would
propably lead to un-balanced partitions as well as complexify code.

If you need to propagate hard links, you have 3 options:

* Re-create hard links on the target, but this is not optimal as you may not
  want to link 2 files together, even if they are similar

* Pre-copy hard linked files together (using find's '-type f -links +1' options)
  before running fpsync. That will work but linked files that have changed
  since your first synchronization will be converted back to regular files when
  running fpsync

* Use a final -monolithic- rsync pass with option -H that will re-create them

SSH options :
-------------

When dealing with SSH options and keys, keep in mind that fpsync uses SSH for
two kinds of operations :

* data synchronization (when ssh is forked by rsync),
  can occur locally or on remote workers (if using any)
* communication with workers (when ssh is forked by fpsync),
  only occurs locally (on the scheduler)

If you need specific options for the first case, you can pass ssh options by
using rsync's option '-e' (through fpsync's option '-o') and triple-escaping
the quote characters :

    $ fpsync [...] -o "-lptgoD -v --numeric-ids -e \\\"ssh -i ssh_key\\\"" \
        /data/src/ login@remote:/data/dst/

The key will have to be present and accessible on all workers.

Fpsync does not offer options to deal with the second case. You will have to
tune your ssh config file to enable passwordless communication with workers.
Something like :

    $ cat ~/.ssh/config
    Host remote
    IdentityFile /path/to/the/passwordless/key

should work.

Limitations :
=============

* Fpart will *NOT* modify data, it will *NOT* split your files !

    As a consequence, if you have a directory containing several small files
    and a huge one, it will be unable to produce partitions with the same size.
    Fpart does magic, but not that much ;-)

* Fpart will not deduplicate paths !

    If you provide several paths to fpart, it will examine all of them. If those
    paths overlap or if the same path is specified more than once, same files
    will appear more than once within generated partitions. This is not a bug,
    fpart does not deduplicate FS crawling results.

* Fpsync only synchronizes directory contents !

    Contrary to rsync, fpsync enforces the final '/' on the source directory. It
    means that directory *contents* are synchronized, not the source directory
    itself (i.e. you will *not* get a subdirectory of the name of the source
    directory in the target directory after synchronization).

Installing :
============

Packages are already available for the following operating systems :

* FreeBSD :       https://www.freshports.org/sysutils/fpart
* Debian :        https://packages.debian.org/fpart
* Ubuntu :        https://packages.ubuntu.com/search?keywords=fpart
* CentOS/Fedora : https://copr.fedorainfracloud.org/coprs/survient/fpart/
* NixOS :         https://nixos.org/nixos/packages.html

If a pre-compiled package is not available for your favourite operating system,
installing from sources is simple. First, if there is no 'configure' script in
the main directory, run :

    $ autoreconf -i

(autoreconf comes from the GNU autotools), then run :

    $ ./configure
    $ make

to configure and build fpart.

Finally, install fpart (as root) :

    # make install

Portability considerations :
============================

On OpenIndiana, if you need to use fpsync(1), the script will need adjustments :

* Change shebang from /bin/sh to a more powerful shell that understands local
  variables, such as /bin/bash.
* Adapt fpart(1) and grep(1) paths (use ggrep(1) instead of grep(1) as default
  grep(1) doesn't know about -E flag).
* Remove -0 and --quiet options from cpio call (they are not supported). As a
  consequence, also remove -0 from fpart options.

On Alpine Linux, you will need the 'fts-dev' package to build fpart(1).

See also :
==========

* See fpart(1) and fpsync(1) for more details.

* Article about data migration using fpart and rsync (GNU Linux Magazine #164 - October 2013, french) :
  http://connect.ed-diamond.com/GNU-Linux-Magazine/GLMF-164/Parallelisez-vos-transferts-de-fichiers

* The partition problem is detailed here :
  http://en.wikipedia.org/wiki/Partition_problem
and here :
  http://en.wikipedia.org/wiki/Bin_packing_problem

* I am sure you will also be interested in :
  https://github.com/jbd/packo
which was developed by Jean-Baptiste Denis as the original proof of concept
See also his newer tool, msrsync :
  https://github.com/jbd/msrsync

They use Fpart or talk about it :
=================================

* Harry Mangalam, from UCI, has an excellent article about data transfer here :
  http://moo.nac.uci.edu/~hjm/HOWTO_move_data.html
Check out his parsyncfp tool (using fpart) here :
  https://github.com/hjmangalam/parsyncfp

* Intel has written a white paper about data migration, presenting fpart and fpsync :
  http://www.intel.com/content/dam/www/public/us/en/documents/white-papers/data-migration-enterprise-edition-for-lustre-software-white-paper.pdf

* Amazon uses fpart and fpsync in their EFS-to-EFS backup solution :
  http://docs.aws.amazon.com/solutions/latest/efs-to-efs-backup/considerations.html
  See the "Amazon Elastic File System (Amazon EFS) for File Storage" presentation here
  (AWS Storage Days, New York, September 6-8, 2017) :
  https://www.slideshare.net/AmazonWebServices/amazon-elastic-file-system-amazon-efs-for-file-storage
  and the Amazon EFS performance tutorial here :
  https://github.com/aws-samples/amazon-efs-tutorial/tree/master/performance
  both presenting fpart and fpsync capabilities.

* Dave Altschuler wrote dsync, a tool using fpart + rsync or rclone that can sync to the cloud :
  https://github.com/daltschu11/dsync

* Bioteam used fpart + fpsync + rsync to migrate 2 PB of data :
  https://www.slideshare.net/chrisdag/practical-petabyte-pushing

* FAS RC (Harvard University) writes about fpsync to move data on Harvard's Odyssey cluster :
  https://www.rc.fas.harvard.edu/resources/documentation/transferring-data-on-the-cluster/#fpsync

* Standford University's Sherlock HPC cluster offers fpart as a file management tool :
  https://www.sherlock.stanford.edu/docs/software/list/

* The QCIF (Queensland Cyber Infrastructure Foundation) offers fpart on QRIScloud :
  https://www.qriscloud.org.au/support/qriscloud-documentation/94-awoonga-software

* Steve French mentioned fpart and fpsync at 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM) :
  https://lwn.net/Articles/789623/

* Microsoft suggests using fpart and fpsync to speed-up file transfers on Azure :
  https://docs.microsoft.com/en-us/azure/storage/files/storage-troubleshoot-linux-file-connection-problems#slow-file-copying-to-and-from-azure-files-in-linux

Author / Licence :
==================

Fpart has been written by Ganael LAPLANCHE <ganael.laplanche@martymac.org>
and is available under the BSD license (see COPYING for details).

Thanks to Jean-Baptiste Denis for having given me the idea of this program !

Contributions :
===============

FTS code comes from FreeBSD :

    lib/libc/gen/fts.c -> fts.c
    include/fts.h      -> fts.h

and is available under the BSD license.
