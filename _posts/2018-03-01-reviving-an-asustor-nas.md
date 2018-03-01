---
layout: post
title:  "Reviving an ASUSTOR NAS"
date:   2018-03-01 12:43:00 +0100
categories: hardware
---

<div style="text-align:center;"><image src="{{ "/assets/2018/2018-03-01-asustor-as202t.png" | absolute_url }}" alt="ASUSTOR AS-202T via their product catalogue" height="250"/></div>

As folk lore tells us, one shall never play on patch day. Apparently the same applies to ASUSTOR
hardware. In this article, I'll talk about the process of recovering an ASUSTOR AS-202T NAS, which
fails with an "Unknown error (Ref. 5052)" during initialization. Initialization is the process
where the NAS's flashed minimal OS installs a full OS on the first partition of the NAS's disks.

To the impatient: If ASUSTOR Control Center shows that your initialization OS image is of version
3.0.0.R8N2 then follow the instructions at the end of this article to fix the error.

# Initial and futile rescue attempts

Googling for the magic words *asustor unknown error ref. "5052"* - 5052 being quoted so that I only
receive results that do indeed include the error code (Google Fu!) yields exactly one result: A
Facebook post with a screenshot of the problem and zero comments and a blog article of a Japanese
writer, who suggests to boot the NAS with only one hard drive during initialization.

Tried that, didn't work. I also tried using different, smaller and older HDDs, initializing the
drives with various partition tables (MBR, GPT, all zeroed out), and using older firmware images
without success.

I concluded that the device's software or some hardware component was shot. With an "unknown error"
it could be anything at this point: The storage controller could fail to write to the disk. The
software could attempt to reach some vendor server that's malfunctioning during the update. A write
error could have occured when the minimal OS image was flashed (or the flash was corrupted,) so that
the updater crashed mid-way. It could also be a software bug caused by insufficient QA. (Spoiler:
That was it.)

If this was a customer's device, it'd tell him at this point to buy a new one and send this one in
to get his money back, because having me take it apart is more expensive. Unfortunately for me, and
fortunately for you my dear reader who might come from Google while looking for a possible solution
to the mysterious error 5052, the device was from my dearest gliding plane club FVA - and the
warrenty had expired. Might this be the reason the club ended up with the device in the first place?

# What's error 5052?

Apparently, the error was so uncommon that:

1. Google didn't know about it
2. ASUSTOR didn't get it often enough to bother documenting it
3. It must happen rarely as implied by (1) and (2)

Let the reverse engineering fun begin. I quickly found out that:

* It's an i686 machine.
* The minimal OS image was derived from version 3.0.0.R8N2. This is visible from the Windows
  software "ASUSTOR Control Center" aswell as from the JSON visible in the initialization page web
  server's HTML source.
* The minimal OS image starts an SSH server. Login as root is possible with password "admin", as
  security best-practices demand.

After connecting to the SSH server I also found that:

* It's built on top of a reasonably modern Linux kernel.
* The entire root-fs comes from an initrd, so make changes at will. Rebooting the device resets the
  system's state.
* The webserver would only retrieve the firmware image, check for a compatible model ID and check
  the md5 of the uploaded/downloaded image. It's not responsible for the error.
* The firmware updater is custom-built by ASUSTOR or one of its contractors and located at
  `/usr/sbin/fwupdate`. It heavily makes use of a proprietary support library at
  `/usr/lib/libnasman.so`. It also uses `/usr/lib/libmfh.so` for writing to the device's internal
  flash. The `fwupdate` executable is what throws the error codes, so that's our target.

You can fetch the firmware updater files either directly from the device via SSH or by finding the
firmware image `AS2XX_3.0.0.R8N2.img` via Google and unpacking it like so:

`tail -n+21 AS-2XX_3.0.0.R8N2.img | tar -x initramfs`.

Then unpack the initramfs using standard methods:

```sh
mkdir initramfs_unpacked
cd initramfs_unpacked
zcat ../initramfs | cpio -idm
```

Before we start: ASUSTOR's official download page for firmware images does not present a license or
contract that limits their use.

Let's throw the `fwupdater` executable at IDA. It's responsible for the high level firmware update
logic if one believes the output of `strings` and thus most likely to contain our error codes. The
responsible chunk of code is quickly identified after scanning over the list of utility functions
and settling for analyzing references to `Set_Error_With_Debug`:

![ASUSTOR firmware updater error 5052]({{ "/assets/2018/2018-03-01-fwupdate-5052.png" | absolute_url }})

The bunch of `jz` instructions above makes it likely that this is a *fallback error code*, which is
hit only if none of the above specific codes matched. This expands our search from a single error
code to a gazillion. Eek.

I conclude that:

* I now need a therapist after reading the `fwupdate` disassembly. I won't post the most terrible
  parts here for your sanity, but suffice to say that execve() on `/bin/sh -c` with `/bin/echo` and
  sh-supported output redirection is not how you write to files in C.
* My issue 5052 comes somewhere from the depths of `fwupdate` and/or `libnasman` and I still can't
  rule out hardware damage.

# The Solution

Since the software doesn't write proper logs and I'm not up for a remote debugging session on the
NAS to find the chunk of code that really fails, I try to instead compare the `fwupdate` and
`libnasman` binaries from the most recent firmware version 3.0.5 to that of 3.0.0. And indeed,
there were some changes in the function whose error code is used above.

Let's transplant 3.0.5's firmware updater onto 3.0.0 and see what happens:

1. Unpack the initramfs of firmware 3.0.5 from ASUSTOR's website
2. Boot the NAS that runs the minimal 3.0.0 firmware image
3. Use `scp` or any SFTP client to transfer the following files from 3.0.5 on top of the in-RAM file
   system of the minimal image:
    * `/usr/sbin/fwupdate`
    * `/usr/lib/libnasman.so.0.0`
4. Ensure all overwritten files are executable
5. Now connect to the web server of the NAS image; upload the 3.0.5 .img file

Lo and behold, the update runs through! Thus, the hardware was fine. It's unlikely that a workaround
for some hardware issue was added in the recent software updates. The hardware is from 2013 after
all.

I conclude that the minimal OS image of firmware 3.0.0 was simply FUBAR. Few people ever noticed
this because the device does not ship with 3.0.0 and the normal update process apparently worked
smoothly enough. Thus, only those who updated to 3.0.0 and wiped their internal disks afterwards
and prior to the release of 3.0.1 two weeks later would have been affected.

# Conclusion

The setup of the product itself after the firmware was fixed went smoothly. However, I'd be careful
with updating the firmware of this device in the future: There is no documented recovery method for
the internal flash image. If an update breaks what's written there, then your device is toast.
That's not a particularly nice design.
