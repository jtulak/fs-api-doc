# FS API doc

FS API is an attempt to unify various filesystem-related tools. The goal is to
have a standardized machine-friendly way of using those tools accross various
filesystems and programs, so scripts doesn't have to do screenscraping.  This
section outlines the intended direction and milestones for this project.  At
this moment, it is in a very early concept phase and gathering ideas.

The full text of the initial proposal raised in mailing list is in the file
``proposal.md``.


## Direction

The most likely approach is to incrementaly implement some small things that
will smooth the roughest corners, while providing the API first as a wrapper
with screenscraping. Later on, FS maintainers can be more easily convinced to
accept the new API, if there already is some usage.

## User Stories
* create/resize/remove/check a filesystem on a volume
* get basic stats (label, uuid, size, free space, blocksize)
* set/edit/unset quotas (disk, user, group, project)
* add/remove device from a pool
* create/delete/mount subvolumes and snapshots in a pool

The reason why I include the pool and subvolume capabilities of btrfs (and
possibly zfs) is that other filesystems can achieve similar results with LVM
and the users of this FS API are tools that are bridging and hiding this
difference.

## Notes

First some generic notes, then categorised.

* JSON output-only helpers without any dependencies from nvme-cli:
  https://github.com/linux-nvme/nvme-cli/blob/master/json.c
* Parted is a tool that was repeatedly mentioned as having a bad output too
* For some tools (fsck, tune2fs, mkfs?) success/failure information should be
  enough
* e2fsprogs has progress bar support: ``e2fsck -C``
* Anaconda had some issues with fs tools that waited for a confirmation. Find
  out more, was this solved correctly, or just workarounded?

### For Integration
LVM has a reporting engine which perhaps could be used to gather all messages
before printing them out in any desired format. Contact Alasdair G Kergon \<agk
at redhat\>


### EVMS
Something similar was attempted before with EVMS by IBM. Their userspace
component has a plug-in architecture so that file systems could provide a
shared library which could be used by GUI and CLI tools.

It was removed in commit ``921f4ad53``: *"Remove support for EVMS 1.x plugin
library"* and perhaps there is something useful.

# License

All documentation in this repository is licensed under a
[Creative Commons Attribution-Share Alike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0), except quoted emails written by other people than jtulak.


