# Proposal

*This email was submitted to linux-fsdevel@vger.kernel.org to start the
discussion: https://www.spinics.net/lists/linux-fsdevel/msg113068.html*

Hi guys

Currently, filesystem tools are not made with automation in mind. So
any tool that wants to interact with filesystems (be it for
automation, or to provide a more user-friendly interface) has to
screen scrape everything and cope with changing outputs.

I think it is the time to focus some thoughts on how to make the fs
tools easier to be used by scripts and other tools. Now, to ease you,
the answer to the obvious question "who will do it" is "me". I don't
want to force you into anything, though, so I'm opening this
discussion pretty early with some ideas and I hope to hear from you
what do you think about it before anything is set in the stone. (For
those who visited Vault this year, Justin Mitchell and I had a talk
about this, codename Springfield.)

The following text attempts to identify issues with using
filesystems-related tools in scripts/applications and proposes a
solution to those issues.

Content:
1. A quick introduction
2. Details of the issues
3. Proposed Solutions
4. Conclusion

## 1. A quick introduction

I discussed this topic with people who are building something around
fs tools. For example, the developer of libblockdev (Vratislav
Podzimek, https://github.com/vpodzime/libblockdev) or system storage
manager (was Lukas Czerner, now it is me,
https://sourceforge.net/projects/storagemanager/), and the listed
issues are a product of experience from working with those tools. The
issues are related mostly to basic operations like mkfs, fsck,
snapshots and resizing. Advanced/debugging tools like xfs_io or xfs_db
are not in the focus, but they might benefit from any possible change
too if included in it.

The main issues of the current state, where all the tools are run with
some flags and options and produce a human readable output:
* The output format can change, sometimes without an easy way of
  detection like a version number.
* Different output formats for different tools even in a single FS,
  thus zero code reuse/sharing.
* Screenscraping can introduce bugs (a rare inner condition can add a
  field into the output and break regular expressions).
* No (or weak) progress report.
* Different filesystems have different input formats (flags, options)
  even for the most basic capabilities.
* Thread safety, forking

## 2. Details of the issues

Let’s look at the issues now: why it is an issue and if we can do some
small change to fix it on its own.


The output format can change, sometimes without an easy way of
detection like a version number. Most filesystems are well behaved,
but still, we don’t know what exactly are people doing with the tools
and even adding a new field can possibly break a script. Keeping a
compatibility with older versions adds another complexity to such
tools.

What can be done about this? The new fields have to be printed somehow
and changing the format of the standard output would break everything.
Making sure that if the input or output changes in any way, it is
always with a detectable difference in the version number is a good
practice, but it doesn’t solve the issue, it only makes hacking around
it easier.

What can really help is to have an alternative output (which can be
turned on when the user wants it), which is easy to parse and which is
resilient to a certain degree of changes because it can express
dynamic items like lists or arrays: JSON, XML...


Different input/output formats for different tools even in a single
FS, thus zero code reuse/sharing: Support for every tool and every
filesystem has to start from a scratch. Nothing can be done about it
without a change in the output format. But if an optional JSON or XML
or something was supported, then instead of creating a parser for
every tool, there could be used just one standard and already a
well-tested library.


Screenscraping can introduce bugs (some rare inner condition can add a
field into the output and break regular expressions): Well, let’s just
look at how many services still can’t even parse and verify an email
address correctly. And we have a lot more complex text… Again, some
easy-to-parse format with existing libraries that would turn the text
into a bunch of variables or an object would help.


No (or weak) progress report: Especially for tools that can run for a
long time, like fsck. Screenscraping everything it throws out and then
deciding whether it is a progress report message (because instead of
“25 %” it says “running operation foo”), a message to tell the user,
or something to just ignore is a lot less comfortable and error prone
than “{level: ‘progress’, stage: 5, msg: ‘operation foo’}”.


Different filesystems have different input formats (flags, options)
even for the most basic capabilities: Similar to “Different
input/output formats for different tools...”. For example, for
labeling a partition, you use ``mkfs.xfs -L label``, but ``mkfs.vfat -n
label``. However, changing this requires getting the same functionality
with a common basic specification to other filesystems too.


Thread safety, forking: The people who work on a library, like
libblockdev, doesn’t like that they have to fork another process over
which they have no control, as they can’t guarantee anything about it.
This can’t be fixed by changing the output format, though, but would
require making a public library providing a similar interface as the
existing fs tools. No detailed access to insides is needed, just a way
how to run mkfs, fsck, snapshots, etc… without spawning another
process and without screenscraping.


## 3. Proposed Solutions

There are two (complementary) ways how to address the issues: add a
structured, machine-readable format for input/output of the tools
(e.g. JSON, XML, …) and to create a library with the functionality of
the existing tools. Let’s look now at those options. I will focus on
what changes they would require, what would be the price for
maintaining that solution and if there are any drawbacks or additional
advantages.

An optional third option would be to create a system service/daemon,
that would use dbus or some other structured interface to accept jobs
and return results. I think that LVM people are working on something
like this.

The proposed solutions are ordered in regards to their complexity.
Also, they can be seen as follow-ups, because every proposed option
requires big part of the work from the previous one anyway.


### 3.1. Structured Output
In other words, what LVM already does with ``--reportformat``
{basic|json}, and lsblk with ``--json``. Possibly, we could also make JSON
input too. That would allow the user to, instead of using all the
flags and options of CLI, make something like
``--jsoninput="{dev:’/dev/abc’, force: true, … }"``

Some preliminary notes about the format:
Most likely, this would mean JSON. JSON is currently preferred over
XML because it is easier to read by humans if the need arises, it’s
encoder/parser is lighter and easier to use. Also, other projects like
LVM, multipath or lsblk already uses JSON, so it would be nice to
don’t break the group.

Required implementation changes/expected problems:
In an ideal world, a simple replacement of all prints with a wrapping
function would be enough. However, as far as I know, an overwhelming
majority of the tools has printing functions spread through the code
and prints everything as soon as it knows the specific bit of
information.

Change of the output into a structured format means some refactoring.
Instead of simple printf(), an object, array or structure has to be
created and rather than pure strings, more diagnostically useful
values have to be added into it in place of the current prints. Then,
when it is reasonable, print it out all at once in any desired format.
The “when reasonable” would usually mean at the end, but it could also
print progress if it is a long time running operation.

Because of the kinds of the required changes, the implementation can
be split into two parts: first, clean the code and move all the prints
out from spaghetti to the end. Then, once this is done, add the
structured format.

Maintaining overhead:
Small. By separating the printing from generating the data, we don’t
really care about the output format anywhere except in the printing
function itself, and if a new field or value is added or an old one
removed, then the amount of work is roughly equal to the current
state.

Drawbacks:
Searching for a place in the code where something happens would be
more complicated - instead of simple search for a string that is the
same in the code and in the output, one would search for the line that
adds the data to the message log. This could be simplified by using
``__LINE__`` macro with debug output. (With JSON, an additional field
would not affect anything, so it would be completely safe.)

Additional advantages:
The refactoring can clean up the code a bit. It is easy to add any
other format in the future. Our own test suite could also benefit from
the structured output.

Comment:
The most bang for the buck option and most of the work done for this
is required also for every other option. In terms of specific
interface (i.e. common JSON elements), we need to identify a common
subset of fields/options that every fs will use and anything else move
into fs-specific extensions.

With regards to the compatibility between different filesystems, the
best way how to specify the format might be a small library that would
take raw data on one side and turn it into JSON string or even print
it (and the reverse, if input supports json too). This way, we would
be sure that there really is a common ground that works the same in
every fs.

Another way how to achieve the compatibility is to make an RFC-like
document. For example: All occurrences of a filesystem identifier MUST
be in a field named 'fsid' which SHOULD contain a UUID formatted
string. I think this is useful even if we end up with the library as a
way to find out the common ground.

### 3.2. A Library
A library implementing basic functions like: ``mkfs(int argc, char
*argv[])``, ``fsck()``, … etc. Once done, binding for other languages like
Python is possible too.

Required implementation changes/expected problems:
If the implementation of this library would follow up after the
changes that add the structured output, then most of the work would be
already done. The most complex issue remaining would probably be that
there can be no exit() call - if anything fails, we have to gracefully
return up the stack and out of the library.

A duplicity of functionality is not an issue because there is none -
the binary tools like mkfs.xfs would become simple wrappers around the
library functions: pass user input to the specified library call, then
print any message created in the library and exit.

Maintaining overhead:
None? The code would be cleaned up and moved, but there wouldn’t be
new things to maintain.

Drawbacks:
I can’t think out any...

Additional advantages:
The refactoring can clean up the code a bit.

Comment:
Useful and nice to have, but doesn’t have to be done ASAP.


### 3.3. A system service
A system service/daemon, that would use dbus or some other structured
interface to accept jobs and return results.

Required implementation changes/expected problems:
We don’t want the daemon to exit if it can’t access a file, we don’t
want it to do printfs(), … So, at the end, we have to do the
structured output and library and then a lot of work above it.

Maintaining overhead:
All the system services things plus whatever the other two solutions require.

Drawbacks:
The biggest maintaining overhead of all proposed solutions, basically
a new tool/subproject. Using dbus in a third party project is more
work than to just include a library and call one function.

Additional advantages:
It wouldn’t be possible to attempt concurrent modifications of a device.

Comment:
In my opinion, this shouldn’t be our project. Let other people make it
their front if they want something like this and instead, just make it
easier for them by using one of the other solutions.


## 4. Conclusion

A structured output is something we should aim for. It helps a lot
with the issues and it is the cheapest option. If it goes well, it can
be later on followed by creating a library, although, at this moment,
that seems a premature thing strive for. Creating a daemon is not a
thing any single filesystem should do on its own.

And thank you for reading all this, I look forward to your comments. :-)
