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

![htop running on iOS 11]({{ "/assets/2017/2017-12-26-ios-htop.png" | absolute_url }})

Due to popular demand I'm bundling up a subset of these utilities. What's inside? All the cool
stuff, that Morpheus does not ship in his binpack yet! Highlights:

* **htop** 2.0.2
* **git** 2.15.1 with full TLS support *(new!)*
* **curl** 7.57.0 using iOS' certificate store *(new!)*
* **rsync** 3.1.2 *(new!)*
* **ag** "The Silver Searcher” 2.1.0
* **tree** 1.7.0 🎄
* **GNU coreutils** 8.26, single-executable build
* Apple binaries, that my oh-my-zsh setup needs to work right (env and alike)
* Apple pcre-9 for ag
* other utilities and symlinks, which are available on macOS High Sierra and make sense on iOS

All macOS system tools are sourced from <https://opensource.apple.com/>.

I'll update the tar file if utilities turn out to not work as expected. I'll also remove utilities
that Morpheus ends up including into his binpack, that is bundled with LiberiOS and available on
[newosxbook.com](http://newosxbook.com/tools/iOSBinaries.html). Ideally, I'll end up shipping an
empty tar file somewhen.[^3]

```
$ tree
.
├── bin
│   ├── [ -> test
│   ├── ag
│   ├── apply
│   ├── basename
│   ├── coreutils
│   ├── curl
│   ├── curl-config
│   ├── dirname
│   ├── echo
│   ├── env
│   ├── expr
│   ├── getopt
│   ├── git
│   ├── git-receive-pack -> git
│   ├── git-shell
│   ├── git-upload-archive -> git
│   ├── git-upload-pack
│   ├── gitk
│   ├── groups -> id
│   ├── htop
│   ├── jot
│   ├── logname
│   ├── mktemp
│   ├── nice
│   ├── od -> hexdump
│   ├── printenv
│   ├── rsync
│   ├── shlock
│   ├── test
│   ├── tree
│   ├── uptime -> w
│   ├── users
│   ├── w
│   ├── whereis
│   ├── who
│   ├── whoami -> id
│   └── yes
├── include
│   ├── curl
│   │   ├── curl.h
│   │   ├── curlver.h
│   │   ├── easy.h
│   │   ├── mprintf.h
│   │   ├── multi.h
│   │   ├── stdcheaders.h
│   │   ├── system.h
│   │   └── typecheck-gcc.h
│   ├── pcre.h
│   ├── pcre_scanner.h
│   ├── pcre_stringpiece.h
│   ├── pcrecpp.h
│   ├── pcrecpparg.h
│   └── pcreposix.h
├── lib
│   ├── libcurl.4.dylib
│   ├── libcurl.dylib -> libcurl.4.dylib
│   ├── libcurl.la
│   ├── libpcre.1.dylib
│   ├── libpcre.a
│   ├── libpcre.dylib -> libpcre.1.dylib
│   ├── libpcre.la
│   ├── libpcrecpp.0.dylib
│   ├── libpcrecpp.a
│   ├── libpcrecpp.dylib -> libpcrecpp.0.dylib
│   ├── libpcrecpp.la
│   ├── libpcreposix.0.dylib
│   ├── libpcreposix.a
│   ├── libpcreposix.dylib -> libpcreposix.0.dylib
│   ├── libpcreposix.la
│   └── pkgconfig
│       ├── libcurl.pc
│       ├── libpcre.pc
│       ├── libpcrecpp.pc
│       └── libpcreposix.pc
├── libexec
│   ├── git-core
│   │   └── ...
│   └── path_helper
├── sbin
│   ├── arp
│   ├── chroot
│   ├── ping6
│   ├── route
│   ├── traceroute
│   └── traceroute6
└── share
    ├── aclocal
    │   └── libcurl.m4
    └── git-core
        └── ...
```

# Download

⚠️ This bundle is meant for developers, who are interested in using the command-line tools listed
above. This download provides zero benefit for the average iOS/Cydia user.

[morebintools64.tar.gz](https://s3.eu-central-1.amazonaws.com/mologie.github.io/assets/morebintools64.tar.gz)
([signature](https://s3.eu-central-1.amazonaws.com/mologie.github.io/assets/morebintools64.tar.gz.sig))  
<small>last updated Dec 27th 18:00 CET: added git, curl, rsync, libpcre; fixed ag, coreutils</small>

Verify the download using GPG
([key on keybase.io](https://keybase.io/mologie/pgp_keys.asc?fingerprint=4f8f50e9df8d0f28a5ee95ae8e7074f534e41872)):

```
$ gpg --verify morebintools64.tar.gz.sig
gpg: assuming signed data in 'morebintools64.tar.gz'
gpg: Signature made Wed Dec 27 17:58:10 2017 CET
gpg:                using RSA key 8E7074F534E41872
gpg: Good signature from "Oliver Kuckertz <oliver.kuckertz@mologie.de>" [ultimate]
gpg:                 aka "Oliver Kuckertz <oliver.kuckertz@rwth-aachen.de>" [ultimate]
gpg:                 aka "Oliver Kuckertz <oliver.kuckertz@softwific.com>" [ultimate]
```

**Read the warnings below,** then copy it to the iDevice and untar to `/jb/usr`:

* Assume, that each utility can and will erase your root file system, eat your first-born and turn
  your iDevice into a fire-breathing dragon, because neither Apple nor I thoroughly tested the tools
  on iOS.
* You must extract the bundle to `/jb/usr` for dynamically linked programs (git, curl, ag, etc.)
  to work correctly. Tinkerers can use `install_name_tool` to change where dylibs are loaded from.
* I intentionally did not set the suid bits for traceroute/traceroute6.

```sh
$ tar -xvf /tmp/morebintools64.tar.gz -C /jb/usr
```

Send me an email (footer) for voting for your favorite tool to be added to the bundle when I next
update it, to let me know if you had success or issues  with it or how you're using it. I may not
respond to all emails, but I sure do read them all. 💌

# Footnotes

[^1]: <http://newosxbook.com/liberios/>

[^2]: iPad Air 1 Cellular

[^3]: There's no point in having two iOS user-space binary distros that cover the same executables, which was pointed out to me after I proposed to set up an APT repository with 64-bit iOS core binaries. This release only complements the binpack from newosxbook.com. Should I draw the wrath of a certain Morpheus with this publication, I humbly request a short note via email.
