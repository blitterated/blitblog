---
date: 2022-07-18T08:59:00Z
title: "Broken Xcode Command Line Tools Installation"
draft: true
image:
   hero: audio-vaporwave-fence.png
   thumb:
author: blitterated
tags: ["XCode", "Homebrew", "CommandLineTools", "xcode-select", "Software Development", "Tooling"]
categories: ["Software"]
---
I got the following error trying to update the [QMK CLI for Mac OS](https://github.com/qmk/qmk_cli#macos) via Homebrew.

```text
==> Installing qmk/qmk/qmk dependency: avr-binutils
xcrun: error: active developer path ("/Applications/Xcode.app/Contents/Developer") does not exist
Use `sudo xcode-select --switch path/to/Xcode.app` to specify the Xcode that you wish to use for command line developer tools, or use `xcode-select --install` to install the standalone command line developer tools.
See `man xcode-select` for more details.
xcrun: error: active developer path ("/Applications/Xcode.app/Contents/Developer") does not exist
Use `sudo xcode-select --switch path/to/Xcode.app` to specify the Xcode that you wish to use for command line developer tools, or use `xcode-select --install` to install the standalone command line developer tools.
See `man xcode-select` for more details.
```

I'd uninstalled XCode ages ago, so I opted to just install the standalone command line developer tools.

```sh
xcode-select --install
```

```text
xcode-select: error: command line tools are already installed, use "Software Update" to install updates
```

WTF...

I check `man xcode-select` for documentation on how to uninstall the command line tools, but there's nothing there. While googling for "reinstall xcode command line tools", I found [this](https://mac.install.guide/commandlinetools/7.html)

It suggests printing out the path to the command line tools

```sh
xcode-select -p
```

```text
/Applications/Xcode.app/Contents/Developer
```

That path matches the active developer path from the original error up above. But look at what happens when we try to list the contents of that directory.

```sh
ls /Applications/Xcode.app/Contents/Developer
```

```text
ls: /Applications/Xcode.app/Contents/Developer: No such file or directory
```

At this point, I just want to wipe out the install and start from scratch. Some setting somewhere is messed up. I followed the link to [Uninstall Xcode Command Line Tools](https://mac.install.guide/commandlinetools/6.html)

AHA! According to that page, Homebrew thinks I have the full install of XCode. With a full install, the CommandLineTools would reside at `/Applications/Xcode.app/Contents/Developer`, which does not exist on my Mac. But when there's no XCode install, the CommandLineTools reside at `/Library/Developer/CommandLineTools` which _does_ exist on my machine. This must be why `xcode-select install` keeps reporting "xcode-select: error: command line tools are already installed"

Look at what happens when we list the contents of where the command line tools _should_ be installed.

```sh
ls /Library/Developer/CommandLineTools
```

```text
Library SDKs    usr
```

It's populated! I'm not sure if that's a full install though. So I'm going to blow it away and start over again.

Blow it all away.

```sh
sudo rm -rf /Library/Developer/CommandLineTools
```

Reinstall the command line tools.

```sh
xcode-select --install
```

Cool! I got the GUI dialogs and it installed. 

Let's see the top level of contents again.

```sh
ls /Library/Developer/CommandLineTools
```

```text
Library SDKs    usr
```

Looks the same. Maybe next time I can just use this:

```sh
xcode-select --switch /Library/Developer/CommandLineTools
```

Tried to upgrade again, and got the same error

```text
xcrun: error: active developer path ("/Applications/Xcode.app/Contents/Developer") does not exist
Use `sudo xcode-select --switch path/to/Xcode.app` to specify the Xcode that you wish to use for command line developer tools, or use `xcode-select --install
```

It looked like it finished OK, but there was nothing new installed. The `qmk` command also no longer exists.

```text
=> Upgrading 1 outdated package:
qmk/qmk/qmk 1.0.0_1 -> 1.1.0
```

```sh
ls /usr/local/Cellar/qmk
```

```text
1.0.0_1
```

Ok, let's try updating that path then. Turns out it needs `sudo`.

```sh
sudo xcode-select --switch /Library/Developer/CommandLineTools
xcode-select -p
```

```text
/Library/Developer/CommandLineTools
```

Should be good to go. Let's try again.

Alright, no errors when `Installing qmk/qmk/qmk dependency: avr-binutils`. Looks like `xcrun` got the correct path to the command line tools.

```text
==> Installing qmk/qmk/qmk dependency: avr-binutils
==> Patching
==> Applying avr-size.patch
patching file binutils/size.c
Hunk #6 succeeded at 395 (offset -2 lines).
Hunk #7 succeeded at 414 (offset -2 lines).
Hunk #8 succeeded at 464 (offset -2 lines).
Hunk #9 succeeded at 909 (offset -2 lines).
Hunk #10 succeeded at 1008 (offset -2 lines).
==> Applying avr-binutils-elf-bfd-gdb-fix.patch
patching file bfd/elf-bfd.h
Hunk #1 succeeded at 21 with fuzz 2.
==> ../configure --prefix=/usr/local/Cellar/avr-binutils/2.38 --libdir=/usr/local/Cellar/avr-binutils/2.38/lib/avr --infodir=/usr/local/Cellar/avr-binutils/2.
==> make
==> make install
🍺  /usr/local/Cellar/avr-binutils/2.38: 159 files, 13.3MB, built in 4 minutes 42 seconds
```

Success!


```sh
ls /usr/local/Cellar/qmk
```

```text
1.1.0
```

```sh
qmk --version
```

```text
1.1.0
```







