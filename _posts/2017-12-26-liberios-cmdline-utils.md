---
layout: post
title:  "LiberiOS! CLI Tools Bundle for iOS 11"
date:   2017-12-26 19:15:00 +0100
categories: apple
---

Habemus Jailbreak, and it's glorious. LiberiOS[^1] jailbreaks iOS 11.0-11.1.2 and worked flawlessly
on my testing iPad4,2[^2] after meddling with the offsets table embedded into the jailbreak
executable. Official support for the iPad4,2 is probably added by now.

Best of all, LiberiOS doesn't completely mangle iOS' Sandbox, meaning that iOS apps won't be able to
do evil and potentially bork one's system. After all, `rm -rf /` could be one lazy iOS developer's
reset-the-app button. On a jailbroken phone without sandbox, such app behaviour is obviously fatal.

In light of the LiberiOS release, I dug out my repository of ported iOS utilities. Htop, originally
compiled for iOS 10, works well after resigning it with `jtool`:

![htop running on iOS 11]({{ "/assets/2017/2017-12-26-ios-htop.png" | relative_url }})

Due to popular demand I'm bundling up a subset of these utilities. What's inside? All the cool
stuff, that Morpheus does not ship in his binpack yet! Highlights:

* **htop** 2.0.2
* **git** 2.15.1 with full TLS support
* **curl** 7.57.0 using iOS' certificate store
* **GnuPG** 1.4.22 *(new!)*
* **rsync** 3.1.2
* **ag** "The Silver Searcher” 2.1.0
* **tree** 1.7.0 🎄
* **GNU coreutils** 8.26, single-executable build
* **tmux** 2.6 *(new!)*
* libevent 2.1.8 for tmux *(new!)*
* Apple binaries, that my oh-my-zsh setup needs to work (env, touch and alike)
* Apple pcre-9 for ag
* other utilities and symlinks, which are available on macOS High Sierra and make sense on iOS

Additionally, the download contains the following programs, which replace programs from the binpack
shipped with LiberiOS:

* **vim** 8.0 (APPL; LiberiOS' vim is incomplete and does not load support files from `/jb`)
* **zsh** 5.4 (shipped version is 2.5 years old and also wants support files)

All macOS system tools are sourced from <https://opensource.apple.com/>.

I'll update the bundle if utilities turn out to not work as expected. I'll also remove system
utilities that Morpheus ends up including into his binpack, that is bundled with LiberiOS and
available on [newosxbook.com](http://newosxbook.com/tools/iOSBinaries.html). Ideally, I'll end up
shipping an empty tar file somewhen.[^3]

# Usage

It's a tar file, and it contains executables and support files. These (with the exception of
statically linked stuff and those, that don't load files from the `share` directory) expect to live
in `/jb/usr`.

There are three mutually exclusive ways how this bundle can be used:

1. Just pick out the binaries (and libraries) you need and copy them to `/jb/usr`. Don't use my vim
   and zsh builds. Safe, simple and reasonably sound. 

2. Untar the full archive to `/jb/usr` via `tar -xf /path/to/the.tar.gz -C /jb/usr`.  
   <b>Caveat:</b> When you rejailbreak, zsh and vim break, because they get overwritten by LiberiOS'
   versions. Write yourself some shell scripts to fix this after rejailbreaking.

3. Modify LiberiOS' binpack yourself and remove duplicates (vim, zsh.) Install this bundle normally
   after jailbreaking with the modified LiberiOS version. This nets you point (2) without the above
   caveat. New caveat:
   When an update for the jailbreak is released, you'll have to patch it again.  
   <small>Patching LiberiOS is simple enough: Get the binpack-tar from its IPA, unpack and modify it
   to your liking, then use tar to bundle it all up.[^4] Use zip to integrate the patched binpack
   back into the IPA.</small>

If there was any warrenty on this pack or LiberiOS, (3) would void it.

# Download

Version: 2 (compare with `/jb/usr/share/mologie/bintools_version`)

⚠️ <b>This bundle is meant for developers,</b> who are interested in using the command-line tools listed
above. This download provides zero benefit for the average iOS/Cydia user and might, if used
improperly, put their device into a state that's not automatically fixed by common tools.

{% comment %}🌐 If you've got the full archive installed, simply run `upgrade_mologie_binpack` to update to the
next version. Users, who pick out binaries, will have to do so by hand again when updating.{% endcomment %}

The standard disclaimer applies: No warrenty implied.

[morebintools64-2.tar.gz](https://mologie.de/~oliver/mologie.github.io/iosbintools64/morebintools64-2.tar.gz) ([signature](https://mologie.de/~oliver/mologie.github.io/iosbintools64/morebintools64-2.tar.gz.sig))  
<small>Last updated Dec 29th 23:10 CET: Added gnupg v1, vim 8.0, zsh 5.4, tmux, libevent, touch, mkfifo</small>

Verify the download using GPG
([key on keybase.io](https://keybase.io/mologie/pgp_keys.asc?fingerprint=4f8f50e9df8d0f28a5ee95ae8e7074f534e41872)):

```
$ gpg --verify morebintools64-2.tar.gz{.sig,}
gpg: Signature made Fri Dec 29 20:12:30 2017 CET
gpg:                using RSA key 8E7074F534E41872
gpg: Good signature from "Oliver Kuckertz <oliver.kuckertz@mologie.de>"
[...]
```

Send me an email (footer) for voting for your favorite tools to be added to the bundle when I next
update it, to let me know if you had success or issues with it or how you're using it. 💌

# Footnotes

[^1]: <http://newosxbook.com/liberios/>

[^2]: iPad Air 1 Cellular

[^3]: There's no point in having two iOS user-space binary distros that cover the same executables and versions, which was pointed out to me after I proposed to set up an APT repository with 64-bit iOS core binaries. Should I draw the wrath of a certain Morpheus with this publication, I humbly request a short note via email.

[^4]: While you're at it, use `xattr -cr .` and `gtar --owner=0 --group=0` to clean the binpack up a bit. `vm_stat` is shipped with iOS, so you may delete it from the binpack. I also ended up merging `/jb/{bin,sbin}` into the respective folders under `/jb/usr`, because there is no point in separating `/bin` and `/usr/bin` like macOS does it for historical reasons.
