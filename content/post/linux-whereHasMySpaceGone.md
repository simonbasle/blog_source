+++
title = "Linux, Where Has my Space Gone?"
date = 2019-04-30T17:38:49+02:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["btrfs", "df", "du", "disk full"]
categories = ["Linux"]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
[header]
image = "headers/detectivePikachu.png"
caption = "[This is not a penguin, this is Detective Pikachu :link:](http://www.detectivepikachumovie.net/ 'copyright Warner Bros')"


# Useful shortcodes:
# {{% toc%}}
# {{% alert note %}} ... {{% /alert %}}
# {{% alert warning %}} ... {{% /alert %}}
# {{< figure src="/img/foo.png" title="Foo!" >}}
# {{< tweet 666616452582129664 >}}
# {{< gist USERNAME GIST-ID  >}}
# {{< speakerdeck 4e8126e72d853c0060001f97 >}}

# Footnotes: [^1] and [^1]: Footnote example.

# You can also use ASCIIDOC! Add the frontmatter below in the body to activate code callouts
# :icons: font
+++

Recently, my Raspberry Pi was complaining that the disk was full. Which was strange because I just recently removed a bunch of files from there, yet `df -h` was reporting 99% usage. But wait, `du -h --summarize` was only reporting a couple gigs used instead of the whole 14Gb...
What gives?

<!--more-->

# Starring at Red Herrings :rabbit:

Once I first noticed the problem, I ran a quick `df -h` to confirm disk usage, which was reported as 99% full.
Oh my!

My first thought was to look for large files or folders running a custom alias based on `du`, but it reported nothing suspicious.
Even stranger, a `du -h --summarize` to get a grand total of the space taken up by files on the full partition only reported about 2Gb of space vs the 14Gb of the partition.
Time for some detective work! :mag_right:

Well, actually... Time for some StackOverflow searching :sweat_smile:

Three possible reasons for this discrepancy came up with a quick search: files in a folder that gets mounted over, deleted large files and running out of inodes.

### Mounts
Inspecting the `fstab` and current `mount` points unfortunately didn't reveal anything.
I was hoping that a bunch of large files had been mistakenly copied to a path that was unmounted at the time, and which a subsequent mount would have shadowed.
Not it :(

### Deleted Orphans
```
lsof | grep deleted
```
Looking for deleted large files didn't reveal any obvious culprit.
A good old restart for good measure didn't appear to do much good, so it was obviously not the issue...

### Running out of inodes
```
df -i
```
Apparently running out of inodes can cause a filesystem to appear full, which can be verified with the above command.
Not it either... :fearful:

At that point I'm starting to scratch my head.

And that's when I notice something that doesn't ring a bell: both `fstab` and `df` shows that the partition's filesystem is of `btrfs` type.

I don't remember choosing this filesystem, but I wasn't really paying that much attention when setting up this particular Pi...
But after looking into it, it turns out that's a pretty good candidate :bulb:

# `btrfs` did it!
This relatively new(ish) filesystem has two features called `subvolumes` and `snapshots` (actually snapshots _are_ subvolumes).

Subvolumes can be mounted recursively, and when unmounted `du` doesn't pick them up.

Snapshots are special subvolumes that can be easily made either manually or **automatically** and capture the state of a subvolume in an incremental way.

This is pretty useful if you want to be able to roll back to a previous state of the system, without consuming too much disk space (thanks to the incremental nature).

Looking at the list of subvolumes in my partition, I quickly realized that somehow there was automatic snapshots made around `apt-get` usage AND on a weekly and daily basis.

Subvolumes (normal ones AND snapshots) can be listed using the following command:

```
btrfs subvolume list -a /
```

{{% alert note %}}
`-a` instructs to list _all_ subvolumes, not only the ones directly under the provided path (here the root `/`)
{{% /alert %}}

I got a list including something like this:

```
ID 4767 gen 253284 top level 274 path <FS_TREE>/storage/@btrfs-auto-snap_weekly-2019-04-23-0854
ID 4823 gen 264859 top level 259 path <FS_TREE>/storage/@btrfs-auto-snap_daily-2019-04-29-1011
```

`btrfs-auto-snap_weekly`... Could it be it? :anguished:

It took be a bit of time, trial and error, but I was able to confirm this was the issue and fix it:

### Finding Out the Size of the Snapshots

Turns out this is not that easy. You need to:

 1. Ensure at least once that `btrfs` **quotas** are turned on: `btrfs quota enable /`
 2. List the sizes by subvolume id using said quotas feature: `btrfs qgroup show /`
 3. Cross link the result to the list of subvolumes, using the subvolume ID in `btrfs subvolume list -a /`

Easier said than done when the qgroup output is like this:

```
qgroupid         rfer         excl
--------         ----         ----
0/4685          0.00B        0.00B
0/4759          0.00B        0.00B
0/4761          0.00B        0.00B
0/4763       32.29MiB     16.00KiB
0/4765          0.00B        0.00B
0/4767       16.00KiB     16.00KiB
0/4819          0.00B        0.00B
0/4820          0.00B        0.00B
0/4821        2.50GiB        0.00B
```

I looked for an easier alternative and found a script that, after inspection, I deemed safe to run on that machine.
But remember kids, don't accept any script from strangers :honey_pot:

See https://ownyourbits.com/2017/12/06/check-disk-space-of-your-btrfs-snapshots-with-btrfs-du/ for the script presentation and content.

With it I was able to confirm that, indeed, I had 4 weekly snapshots that were in the 2Gb-6Gb range, each.

### Deleting the Snapshots
Snapshots and subvolumes can supposedly be deleted like any other file, but it is better to delete them via `btrfs` which is way faster.

The command is:
```
btrfs subvolume delete NAME_OF_SNAPSHOT
```

But I was confused, because I was unable to find a way to perform that command without a "file doest not exist" kind of error...

### s/Deleting/Mounting/ the Snapshots

In order to be able to find the subvolumes listed by the script above, I needed to first mount the root subvolume somewhere on my current filesystem.
That is, after finding the `/dev/file` corresponding to my partition, of _course_ :confused:

```
mount /dev/mmcblk0p8 /mnt -o subvol=/
cd /mnt/storage
```

This revealed the `storage/` subfolder under the mount point `/mnt/`, in which lay all of my snapshots :tada:

After several `btrfs subvolume delete` commands of the above, unmounting via `umount /mnt` for good measure and running `df -h` again, everything was finally back to normal:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p8   14G  3.5G   11G  26% /
```

So Today I Learned `btrfs`... the hard way :wink: