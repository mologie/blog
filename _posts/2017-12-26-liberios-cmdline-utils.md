---
layout: post
title:  "LiberIOS hype train, htop on iOS"
date:   2017-12-26 19:15:00 +0100
categories: apple
---

Habemus Jailbreak, and it's glorious. LiberIOS[^1] jailbreaks iOS 11.0-11.1.2 and worked flawlessly
on my testing iPad4,2[^2] after meddling with the offsets table embedded into the jailbreak
executable. Official support for the iPad4,2 is probably added by now.

Best of all, LiberIOS doesn't completely mangle iOS' Sandbox, meaning that iOS apps won't be able to
do evil and potentially bork one's system. After all, `rm -rf /` could be one lazy iOS developer's
reset-the-app button. On a jailbroken phone without sandbox, such app behaviour is obviously fatal.

In light of the LiberIOS release, I dug out my repository of ported iOS utilities. Htop, originally
compiled for iOS 10, works flawlessly after resigning it with `jtool`:

![htop running on iOS 11]({{ "/assets/2017/2017-12-26-ios-htop.png" | absolute_url }})

Due to popular demand I'm bundling up a subset of these utilities. What's inside? All the cool
stuff, that Morpheus does not ship in his binpack yet! Highlights:

* htop (the one and only)
* The Silver Searcher (ag)
* tree 🎄
* GNU coreutils, single-executable build
* various binaries that my oh-my-zsh setup needs to work right (env and alike)
* other utilities and symlinks, which are available on macOS High Sierra and make sense on iOS

All tools are sourced from <https://opensource.apple.com/>.

I'll update the tar file if utilities turn out to not work as expected. I'll also remove utilities
that Morpheus ends up including into his binpack, that is bundled with LibreIOS and available on
[newosxbook.com](http://newosxbook.com/tools/iOSBinaries.html). Ideally, I'll end up shipping an
empty tar file somewhen.[^3]

Caveats:

* I intentionally did not set the suid bits for traceroute/traceroute6.
* You may to move ping6 to `/jb/sbin` or `/sbin` to mirror macOS 100%, but it's technically
  irrelevant whether it appears there or not (unless some program invokes it by its absolute path.)
* Something something disclaimer for breaking your iOS device: Assume, that each utility can and
  will erase your root file system, eat your first-born and turn your iDevice into a fire-breathing
  dragon, because neither Apple nor I thoroughly tested the tools on iOS.

```
$ tree
.
├── bin
│   ├── [ -> test
│   ├── ag
│   ├── apply
│   ├── basename
│   ├── coreutils
│   ├── dirname
│   ├── echo
│   ├── env
│   ├── expr
│   ├── getopt
│   ├── groups -> id
│   ├── htop
│   ├── jot
│   ├── logname
│   ├── mktemp
│   ├── nice
│   ├── od -> hexdump
│   ├── printenv
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
├── libexec
│   └── path_helper
└── sbin
    ├── arp
    ├── chroot
    ├── ping6
    ├── route
    ├── traceroute
    └── traceroute6
```

# Download

⚠️ This bundle is meant for developers, who are interested in using the command-line tools listed
above. This download provides zero benefit for the average iOS/Cydia user.

[morebintools64.tar.gz]({{ "/assets/morebintools64.tar.gz" | absolute_url }})
([signature]({{ "/assets/morebintools64.tar.gz.sig" | absolute_url }}))  
<small>last updated: never</small>

Verify using GPG ([key on keybase.io](https://keybase.io/mologie/pgp_keys.asc?fingerprint=4f8f50e9df8d0f28a5ee95ae8e7074f534e41872)):

```
$ gpg --verify morebintools64.tar.gz.sig
gpg: assuming signed data in 'morebintools64.tar.gz'
gpg: Signature made Tue Dec 26 19:19:38 2017 CET
gpg:                using RSA key 8E7074F534E41872
gpg: Good signature from "Oliver Kuckertz <oliver.kuckertz@mologie.de>" [ultimate]
gpg:                 aka "Oliver Kuckertz <oliver.kuckertz@rwth-aachen.de>" [ultimate]
gpg:                 aka "Oliver Kuckertz <oliver.kuckertz@softwific.com>" [ultimate]
```

Copy it to the iDevice, untar anywhere as root. `/jb/usr` sounds right:

```sh
$ tar -xvf /tmp/morebintools64.tar.gz -C /jb/usr
```

# Footnotes

[^1]: <http://newosxbook.com/liberios/>

[^2]: iPad Air 1 Cellular

[^3]: There's no point in having two iOS user-space binary distros that cover the same executables, which was pointed out to me after I proposed to set up an APT repository with 64-bit iOS core binaries. This release only complements the binpack from newosxbook.com. Should I draw the wrath of a certain Morpheus with this publication, I humbly request a short note via email.
