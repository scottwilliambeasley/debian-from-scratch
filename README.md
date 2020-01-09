# debian-from-scratch

An instruction manual for teaching Linux From Scratch users how to make a Debian system.

## Why Debian from Scratch?

The original [Linux from Scratch](https://en.wikipedia.org/wiki/Linux_From_Scratch) manual is purposefully vague as to what technique one should use to manage software dependencies. The suggestions that it gives, while no doubt being interesting exercises in package management, are not necessarily heavy-duty answers to a system administrator who intends on managing his time efficiently. 

The disadvantage of compiling everything to create a fully-fledged system is time. After one builds an LFS system for the first time, he/she is apt to realize that managing dependencies can be an arduous task, to say the least. Going through the insufferable exercise of hunting down dozens to possibly hundreds packages, mapping dependencies, configuring and installing these dependencies in the correct order, just to install a single piece of software, is not a viable alternative to a system administrator who values his time.

The answer to this problem, is obviously to use a package manager. There are many available package managers, the most popular of which are the Debian-based package managers (dpkg and apt), and the Red Hat based package managers (rpm and yum). 

This manual will teach you how to build your system utilizing the Debian set of package management tools by utilizing the temporary system environment created in Linux From Scratch.

## Purpose for this project

I chose to make this manual because I have seen woefully old guides on the internet teaching others how to get dpkg and apt running on their own custom Linux, and people asking on various forums on how to install dpkg and apt, but not getting the help that they need. These guides are outdated and no longer contain up-to-date information, which I intend to fix here in this manual.

This project intends to be a community resource to help those interested in creating their own custom system from the ground up, while fully taking advantage of the Debian suite of package management, dpkg and apt, in order to solve the problems of package dependency installation and management.

## How to use this manual? 

This manual is designed to be used after completing all instructions up to the end of Chapter 5, and some parts of Chapter 6 of the [Linux From Scratch book, version 7.9](http://www.linuxfromscratch.org/lfs/view/7.9/). One first follows the instructions of the original LFS book, and builds the temporary system that is created in LFS Chapter 5. One is required to have a fully-functional temporary system which the the outcome of Chapter 5. 

After completing the preliminary preparations, one then consults this manual and follows it step-by-step.

Like the original LFS manual, when dealing with packages to be compiled, each section already assumes that you have extracted the source code and have changed your main directory into the main folder of the extracted content. However, when dealing with .deb files no such extraction is needed. One only needs to follow the instructions while having the .deb file in your current directory.

## Overview of our Method for Building a Custom Debian System

In the original Linux From Scratch book, we created a cross-toolchain, using the toolchain native to our system. We then used this cross-toolchain to create a native toolchain, which ended up being the temporary '/tools' system environment. This was Chapter 5's goal. We then used this temporary system to build our final system, which was Chapter 6's goal.

In Debian From Scratch, we branch off from the end of Chapter 5. Instead of using the toolchain and other utilies installed in `/tools` to compile every single part of the final system, we instead use this toolset to compile and install Debian's package manager, dpkg  as the first part of our final system. 

We then do some dependency hacking to satisfy all remaining dependencies needed to install apt. This allows us to rely on apt for the overwhelming majority of tasks involving the installation of software onto our new system, and to allow us to avoid the exercise in tedium that is manual dependency management.

We then use apt to install all base packages functionalities needed for the system to operate properly, in the correct order, to prevent problems and broken packages.

## Obtaining all needed packages

Let's start off by using our host system to download the packages we need, and place them somewhere inside our $LFS partition.

#### source files

One of the requirments for DPKG is PKG-Config. This is built in Chapter 6.23 of Linux from Scratch. Be sure to downdown this source as listed in Linux from Scratch. 

[pkg-config](http://pkgconfig.freedesktop.org/releases/pkg-config-0.29.tar.gz)

Link to DPKG

[dpkg](http://http.debian.net/debian/pool/main/d/dpkg/dpkg_1.17.27.tar.xz)

#### .deb files

You will need to download the following .deb files and place them all in the same directory inside of your `$LFS` partition. Only these .deb files should occupy this directory. These are needed to install the entire dependency chain of `apt`. Open the link, and manually download the .deb file corresponding to your LFS system architecture:

[apt](https://packages.debian.org/jessie/apt)

[debian-archive-keyring](https://packages.debian.org/jessie/debian-archive-keyring)

[dpkg](https://packages.debian.org/jessie/dpkg)

[gcc-4.9-base](https://packages.debian.org/jessie/gcc-4.9-base)

[gnupg](https://packages.debian.org/jessie/gnupg)

[gpgv](https://packages.debian.org/jessie/gpgv)

[libacl1](https://packages.debian.org/jessie/libacl1)

[libapt-pkg4.12](https://packages.debian.org/jessie/libapt-pkg4.12)

[libattr1](https://packages.debian.org/jessie/libattr1)

[libbz2-1.0](https://packages.debian.org/jessie/libbz2-1.0)

[libc6](https://packages.debian.org/jessie/libc6)

[libgcc1](https://packages.debian.org/jessie/libgcc1)

[liblzma5](https://packages.debian.org/jessie/liblzma5)

[libpcre3](https://packages.debian.org/jessie/libpcre3)

[libreadline6](https://packages.debian.org/jessie/libreadline6)

[libselinux1](https://packages.debian.org/jessie/libselinux1)

[libstdc++6](https://packages.debian.org/jessie/libstdc++6)

[libtinfo5](https://packages.debian.org/jessie/libtinfo5)

[libusb-0.1-4](https://packages.debian.org/jessie/libusb-0.1-4)

[multiarch-support](https://packages.debian.org/jessie/multiarch-support)

[readline-common](https://packages.debian.org/jessie/readline-common)

[tar](https://packages.debian.org/jessie/tar)

[zlib1g](https://packages.debian.org/jessie/zlib1g)


## Creating the Debian From Scratch system

All following commands need to be performed as `root`, so become the `root` user on your host system:

`su`

### Preparing the virtual kernel filesystem mount points
First we create the directories which are supposed to contain virtual kernel filesystems. These are filesystems which are located in memory only and created dynamically every time the kernel is loaded. 

Each kind has a different purpose. The `devpts` contains device files for each pseudo terminal on your system. The `proc` contains information about every single process. The `sysfs` contains driver and device information. The `tmpfs` is a freely usable space which programs may use to store information in memory. 

Since we have not built our kernel yet, we are forced to use the ones existing on our host system by mounting them in the appropriate locations in our target system. When our system is fully built, the new kernel will automatically mount these filesystems in their appropriate places.
```
mkdir -pv $LFS/{dev,proc,sys,run}
mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3
mount -v --bind /dev $LFS/dev
mount -vt devpts devpts $LFS/dev/pts -o gid=5,mode=620
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```

### Entering our chroot environment

We must now, as the `root` user on our host system, enter our base environment by changing our root directory into the final system's root directory, and use the temporary environment we've previously constructed to build our final system. Use the following command after you have become `root` on your host:

```
chroot "$LFS" /tools/bin/env -i \
HOME=/root                  \
TERM="$TERM"                \
PS1='\[\033[01m\][ \[\033[01;34m\]\u@\h\[\033[00m\]\[\033[01m\]]\[\033[01;32m\]\w\[\033[00m\]\n\[\033[01;34m\]$\[\033[00m\]> ' \
PATH=/bin:/usr/bin:/sbin:/usr/sbin:/tools/bin:/tools/sbin \
/tools/bin/bash --login +h
```

Once in chroot enter the following commands:


```
mkdir /bin
ln /tools/bin/bash /bin/bash
ln /tools/bin/bash /bin/sh
```


### Building PKG-Config

 Build [PKG-Config](http://www.linuxfromscratch.org/lfs/view/7.9/chapter06/pkg-config.html) as show in 6.23 of Linux From Scratch then continue on. 


### Installing dpkg

With our `/tools` environment completely set up, we are ready to directly compile and install `dpkg` into our target environment. Replace the `build` variable with the appropriate architecture if it isn't 64-bit (which I am assuming that it is):


```
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --build=x86_64-unknown-linux-gnu
make
make install
```

##### Creating dpkg's database
We need to create `dpkg`'s database, which is merely a text file located in `/var/lib/dpkg/status`. `dpkg` stores all of its package information in this file, including package version, architecture, dependencies, etc. It does not yet currently exist. Without this file, dpkg will not function correctly, so it is important that we create this before we move forward.

`touch /var/lib/dpkg/status`

##### Creating temporary links linking to /tools/bin/bash
In order the for pre and post-installation scripts that come inside a standard .deb file to work, these need to be able to have access to a shell. These typically either specify using `/bin/sh` or `/bin/bash`. Without access to a shell at the exact location that the script specifies, the installation script will fail thus causing the installation of the package itself to fail.

We must mitigate this problem by making sure that the `/bin` directory already exists, and then creating symlinks from these two locations towards the bash we have in our temporary `/tools` environment. 

When we get to the point of installing debian's "priority:essential" packages, which include both of these shells, these symlinks will be overwritten with native copies of these binaries.



### Installing apt

Before we can install `apt`, and use this to automatically install the most of the rest of our system software, we have to install its immediate dependencies on our target system first.

![](https://cdn.rawgit.com/scottwilliambeasley/debian-from-scratch/master/images/apt-dependencies.svg)
##### Figure 2 - The apt dependency tree, one level deep

Each of these immediate dependencies has their own set of dependencies to fulfill. We shall start by completing the dependency tree for `debian-archive-keyring`. Unlike the compilation process needed to install `dpkg`, the process we now use to install software is by installing .`deb` files using `dpkg`. 

#### Installing debian-archive-keyring

So we shall start fulfilling `dpkg`'s dependencies by first by installing `debian-archive-keyring`. However, there is a problem here - we have a circular dependency as illustrated below.

![](https://cdn.rawgit.com/scottwilliambeasley/debian-from-scratch/master/images/debian-archive-keyring-deps.svg)
##### Figure 3 - The debian-archive-keyring dependency tree, circular dependency in red

`libgcc1` depends on `multiarch-support`, which depends on `libc6`, which depends on `libgcc1`. This is a major problem, as because none of these packages will fully install without having the others.

In order to solve this seemingly impossible problem, we will need to bend the rules a little bit.

First, we need to install `gcc-4.9-base`:

`dpkg -i (location of gcc-4.9-base)`

Then, we will need to install one package first in a partial way

`dpkg - i (LOCATION_OF_libgcc1)`

and tweak the database to convince it that it -was- fully installed:

`sed -ir 's/not-installed/installed/' /var/lib/dpkg/status`

Furthermore, since the package's database entry wasn't updated with the description,maintainer, and version info, we need to append these to the package's database entries. This will prevent `dpkg` from moaning and groaning:

`sed -ri 's/(Package: libgcc1)/\1\nDescription: test\nMaintainer: test\nVersion: 2.19/' /var/lib/dpkg/status`

Now install the other two packages in order:

`dpkg -i (location_of_multiarch)`
`dpkg -i (location_of_libc6)`

And reinstall libgcc1 to cover over our ugly little hack and complete the full installation of each package:

`dpkg -i --reinstall (location of libgcc1)`

#### Installing the rest of apt's dependencies

At this point, I am assuming that -all- of the .deb files you need to install have been placed in a single directory. Double check to see that you have all of these files. `cd` to this directory right now.

The truth is, for the remaining dependencies, the dependency tree for all of them is just too complicated for me to map out and give you a granular series of installation commands to execute what should otherwise be a straightforward operation.

Regardless of the actual dependency tree, there is a quick and dirty way to install the rest of the entire apt dependency tree. Simply execute the following command, and repeat it as many times as you need until dpkg no longer complains:

` dpkg -i *`

What happens here is that dpkg will attempt to install all software packages in the directory. It will inevitably fail, but do not fret. Simply repeat the command until everything is installed.

##### How this works

What happens when we run the command above is that dpkg is attempting to install each package without actually taking into mind the dependency tree. So with each execution of the above command, another level of the dependency tree will be fulfilled, until it's successfully able to confirm that the entire tree was installed. Once you no longer get any errors from dpkg, it means everything was installed.

dpkg is simply not as intelligent as something like apt, which would automatically build a dependency map, and install all prerequisite software before attempting to install software dependent on said software.

You might also notice that dpkg itself is part of the .deb files that needed to be installed. Don't worry, this doesn't break anything. We already have dpkg, but are just installing the official package to update its own database because the database never actually contained itself as part of the list of installed packages.
 
##### Double checking if everything is installed
 
To be absolutely certain, you can execute the following command, which counts how many packages dpkg counts as installed in its database:

`echo $(($(dpkg -l | wc -l)-5))`

If this returns the value 23, then you can correctly assume that everything was installed as intended. 

### Creating networking configuration files

Before we proceed with updating `apt's` cache, we need to define both the system's networking configuration files, and apt's list of software repositories.

#### /etc/resolv.conf

`/etc/resolv.conf` is a file needed in order for your system to have DNS resolution. Without this file, no resolution typically happens, which makes updating `apt`'s cache of installable packages via `/etc/apt/sources.list` difficult.

```
cat > /etc/resolv.conf << "EOF"
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF
```

#### /etc/apt/sources.list

sources.list is a file which `apt` uses to contact the repositories that hold your software. You want to use the repositories from one and only one distribution as much as possible, otherwise you risk breaking your Debian's clear chain of dependencies and creating an insane mess of your system. 

```
cat > /etc/apt/sources.list << "EOF"
# Debian Jessie main repos
deb http://httpredir.debian.org/debian/ jessie main  
deb-src http://httpredir.debian.org/debian/ jessie main  

#Debian Jessie security repos
deb http://security.debian.org/ jessie/updates main  
deb-src http://security.debian.org/ jessie/updates main  

# non-free plugins
deb http://http.debian.net/debian/ jessie non-free contrib main  

# jessie-updates, previously known as 'volatile'
deb http://httpredir.debian.org/debian/ jessie-updates main  
deb-src http://httpredir.debian.org/debian/ jessie-updates main
EOF
```

#### /etc/hosts

`/etc/hosts` is a file used to contain mapping of IP addresses to hostnames. This file is usually checked before DNS queries, at the very least, this should contain your `ipv4` and `ipv6` loopback addresses.

```
cat > /etc/hosts << "EOF"
127.0.0.1       localhost

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
EOF
```

#### /etc/hostname

`/etc/hostname` is used to contain your host's DNS name. Edit this if you like.

```
cat > /etc/hostname << "EOF"
debianfromscratch
EOF
```

### Updating apt's package lists

We are now ready to update our list of packages and take full advantage of `apt`. To do this, we update our local keyring of valid Debian developer gnu pgp signatures using `apt-key update`, and then update with `apt-get update`.
```
apt-key update
apt-get update
```

### Creating user and group databases

Before we install more software, we must make sure that our password, group and authentication mechanisms are all in place. This is because some packages will require the adding of a new user or group to the system as part of their installation process. Without these base functionalities already in place, installation of said packages will fail.

##### debianutils
We install debianutils to provide the `tempfile` command needed by one of `base-passwd`'s installation scripts. Without this command, installation of `base-passwd` will fail.

`apt-get install debianutils`

##### base-passwd 
We install `base-passwd` to provide standard the standard minimal `/etc/passwd` and `/etc/group` files, which are the same across all debian systems. It does this by running the `update-passwd` binary upon its installation.

`apt-get install base-passwd`

##### Creating /etc/shadow and /etc/gshadow
We have to manually create `/etc/shadow` and `/etc/gshadow`, as the `passwd` package will fail to configure if it cannot find these files:

`touch /etc/shadow /etc/gshadow`

##### login 
We then install the `login` package, which gives us the ability to establish new sessions on the system with `login`, privilege escalation with `su`, the linux pluggable authentication module (PAM) files for both said binaries, a fake shell `/bin/nologin`,  and the `/etc/login.defs` file which is essential for group creation. There are more functionalities included with this package, but these are the most mentionable.

`apt-get install login`
 
##### passwd 
We then install `passwd` package, which provides the lion's share of utilities and configuration files used to create and manipulate user and group account information.

`apt-get install passwd`

##### adduser 
We must also install the `adduser` package, because this provides us with the default `/etc/adduser.conf` file which will be needed to install new users properly.

`apt-get install adduser`

##### Establishing root password and shadowfile entries
With all the aforementioned utilities and packages installed, our system is now capable of the full functionality of user & group account manipulation.

At this point, we should run passwd to change our root password. 

`passwd root`

We should then run `pwconv` to convert our /etc/passwd entries into shadow entries in `/etc/shadow`:

`pwconv`

### Fixing the terminal and adding reading/editing utilities

Our terminal does not yet have the full functionality one would expect of a terminal, and standard terminal utilities may still fail to function properly at this point. Let's install the proper libraries and create the configurations needed to make these , before we install 

#### Creating /etc/inputrc file

`/etc/inputrc` is the global configuration file for the used by the `libreadline6` library, which most shells use in order to understand how to handle many special keyboard situations, such as what behavior should be default when hitting the HOME and END keys. Without this file, many special keys and two-key stroke combos such as ctrl+left will fail to work. 

Since this file is not created by default when installing the `libreadline6` library, we must create it ourselves.

```
cat > /etc/inputrc << "EOF"
# /etc/inputrc - global inputrc for libreadline
# See readline(3readline) and `info rluserman' for more information.

# Be 8 bit clean.
set input-meta on
set output-meta on

# To allow the use of 8bit-characters like the german umlauts, uncomment
# the line below. However this makes the meta key not work as a meta key,
# which is annoying to those which don't need to type in 8-bit characters.

# set convert-meta off

# try to enable the application keypad when it is called.  Some systems
# need this to enable the arrow keys.
# set enable-keypad on

# see /usr/share/doc/bash/inputrc.arrows for other codes of arrow keys

# do not bell on tab-completion
# set bell-style none
# set bell-style visible

# some defaults / modifications for the emacs mode
$if mode=emacs

# allow the use of the Home/End keys
"\e[1~": beginning-of-line
"\e[4~": end-of-line

# allow the use of the Delete/Insert keys
"\e[3~": delete-char
"\e[2~": quoted-insert

# mappings for "page up" and "page down" to step to the beginning/end
# of the history
# "\e[5~": beginning-of-history
# "\e[6~": end-of-history

# alternate mappings for "page up" and "page down" to search the history
# "\e[5~": history-search-backward
# "\e[6~": history-search-forward

# mappings for Ctrl-left-arrow and Ctrl-right-arrow for word moving
"\e[1;5C": forward-word
"\e[1;5D": backward-word
"\e[5C": forward-word
"\e[5D": backward-word
"\e\e[C": forward-word
"\e\e[D": backward-word

$if term=rxvt
"\e[7~": beginning-of-line
"\e[8~": end-of-line
"\eOc": forward-word
"\eOd": backward-word
$endif

# for non RH/Debian xterm, can't hurt for RH/Debian xterm
# "\eOH": beginning-of-line
# "\eOF": end-of-line

# for freebsd console
# "\e[H": beginning-of-line
# "\e[F": end-of-line

$endif
EOF
```

#### ncurses libraries and binaries
A large number of command line utilites rely on the ncurses library to provide a text-interface for user interaction over the terminal. These include simple utilities such as `less` and `nano`. Without this library, these utilities will fail to display properly. We must install the complete suite of the ncurses library in order to prevent said errors from occurring:

`apt-get install ncurses-base ncurses-bin ncurses-doc`

#### dialog
Dialog is a perl module which some scripts attempt to use to provide a text-interface used during configuration or installation. You may have noticed some packages warning you that this utility was non-existent during installation. Let's fix this: 

`apt-get install dialog`

#### less, vim and nano
Now that we've installed most of the libraries and utilities needed for terminal utilities, let's install some of the most basic and well-used ones:

`apt-get install less vim nano`

### Creating the standard filesystem hierarchy on a Debian system
Let's create the standard folder structure for a debian system. This can be done easily by installing `base-files`. First, we have to remove the `/var/mail` directory though or else it will complain that it already exists.

```
rm -rf /var/mail
apt-get install base-files
```

### Building the man documentation system
Any Linux system typically has a database full of manual pages, accessed by the `man` command. Most programs we've installed already have already added their documentation files in the proper location. All we need to do now is actually install `man` to take advantage of them, and any more that are added as time passes by.

`apt-get install man`

### Installing all remaining essential packages
We've installed most of the packages, all that is left to do is install the rest of the packages marked with the priority `essential` by the Debian maintainers. Some of these are absolutely essential to system management, and some will barely be used at all. 

To comply with the Debian standard, we must install all of these:

`apt-get install bash bsdutils coreutils dash diffutils e2fsprogs findutils grep gzip hostname libc-bin init mount perl-base sed sysvinit-utils tar util-linux`

Here follows is a short description of each package installed:

| Description | What it provides |
| ------------- | :-----: |
| **bash**: | The gnu bourne-again shell, which is your standard linux shell. |
| **bsdutils**: | Provides a few binaries, most notably `renice` which is needed for changing process priorities, and `logger` which is used for interacting with the syslog system module.|
| **coreutils**: | The absolute most essential group of binaries needed to make any shell useful. |
| **dash**: | The Debian Almquist shell, which is a faster version of sh intended mainly for use by scripts. |
| **diffutils**: | Provides utilities for comparing the contents of files between each other.  |
| **e2fsprogs**: | Provides  utilities for working with the ext family of filesystems.  |
| **findutils**: | Provides the find utility for finding files.  |
| **grep**: | Provides the grep utility, used for finding strings within files or output you pipe into it. |
| **gzip**: | Provides the gzip utility, used for working with files using LZ77 encoding. |
| **hostname**: | Provides the a set of utilities for manipulating the system's host name. |
| **libc-bin**: |  Provides the GNU implementation of the standard C library. Essential for creating and using programs. |
| **init**: | Provides the standard system initialization suite for Debian. |
| **mount**: | Provides the standard system utilities for mounting and unmounting filesystems, including swapfiles.  |
| **perl-base**: | Provides the perl programming language.  |
| **sed**: | Provides the sed programming language, generally used for editing text.  |
| **sysvinit-utils**: | Provides system-v like utilities. |
| **tar**: | Provides the tar program, used for storing and retrieving files from a taped archive.  |
| **util-linux**: | Provides many vital system utilities. |

### Installing the kernel

You have two options here: either you can install the latest kernel image for your system architecture provided by the Debian project, or you can compile your own.

#### Installing Debian's kernel

If you want to install Debian's standard kernel, you will want to search for the available images provided for your architecture, and then install the approriate image.

```
apt-cache search linux-image
apt-get install (selected image)
```

#### Compiling and installing your own custom kernel

You can also compile your own custom kernel if you feel like it. 

To do this, you will need a tarball of the Linux kernel source, which you probably already have.

You will also need to install the gcc package (which incidentally installs most other software needed to compile), the libncurses5-dev package (which provides the libraries needed to use the command line configuration utility for the kernel source), the bc package (a language that supports precision numbers), and the make package (the make utility used for compiling the source into binary). 

`apt-get install gcc libncurses5-dev bc make`

Now open up your extracted kernel source. Ensure that there are no stale files left behind from the developers in the source tree:

`make mrproper`

Install the header files for this particular kernel. You will need these in the future if you intend to compile software that will take advantage of this kernel's API in the future:

```
make INSTALL_HDR_PATH=dest headers_install
cp -rv dest/include/* /usr/include
```

I recommend that you use the default configuration as the base for your kernel build, as it will at the very least ensure that your system will be able to boot using the image we create later.

`make defconfig`

Then, customize your kernel according to your heart's desire.

`make menuconfig`

This book cannot help you figure out what exact modules to add to the kernel - you need determine the exact hardware that exists on your system yourself, and do research as to what modules will make those pieces of hardware functional. Google is your friend. 

After you've customized your kernel, compile it.

`make`

If you've created a modular kernel, install the compiled modules into it:

`make modules_install`

Copy the completed kernel into the /boot/ directory, and make sure that it's name starts with 'vmlinuz'. It's a good idea to append an identifying string to the name of this file. The version number of this kernel will do. We also copy the System.map file, and the config for this kernel (so we can easily examine how this kernel was built). We append our identifying string to this too, because as time passes by we may want to install more kernels.

```
cp -v arch/x86/boot/bzImage /boot/vmlinuz-(identifier)
cp -v System.map /boot/System.map-4.4.2
cp -v .config /boot/config-(identifier)
```

### Adding modular functionality to the system

Regardless of if you installed the standard Debian GNU/Linux kernel, or compiled your own, you are going to need to provide your system with the binaries needed to load, remove and manipulate kernel modules. 

`apt-get install kmod `

The only exception to this case would be if you compiled your own monolithic kernel with no modules, and do not intend on adding any beyond the ones you've already compiled into the kernel (which would generally be the case if you're doing embedded development).

### Making the system bootable


#### Reconfiguring a drive with a pre-installed GRUB2 bootloader
If this Debian Linux system is on a partition of an already bootable drive, all one needs to do to make this system bootable is to replace the configuration file of the drive's bootloader to include this system's corresponding partition in its list of bootable partitions.

If using GRUB2, then using the `grub2-mkconfig` utility makes it very simple to do so. Merely point it to the location of the already existing grub.cfg file:

`grub2-mkconfig -o /boot/(path to grub.cfg)`

#### Installing GRUB2 onto a drive with no pre-installed bootloader
In the event that the partition is on a drive that does not yet have its own bootloader, you must install it and configure it yourself. To install GRUB2, you must first exit your chroot.

```
exit
grub2-install /dev/(location of drive that holds this partition)`
```

Since this GRUB2 install does not have a configuration file yet, let's create one via template:

```
cat > /boot/grub/grub.cfg << "EOF"
# Begin /boot/grub/grub.cfg
set default=0
set timeout=5

insmod ext2
set root=(hd0,2)

menuentry "Debian from Scratch GNU/Linux" {
        linux   /boot/(kernel-location) root=/dev/sda2 ro
}
EOF
```

Then, edit this file to replace the (kernel-location) string in the file with the actual location of said kernel inside the partition.

Afterwards, use the previously aforementioned `chroot` command provided in the 'Entering our chroot' section to return to the chroot.

### Creating /etc/fstab

Before you boot into your system, it is absolutely -vital- for you to create and configure your /etc/fstab file. 

If you do -not- have an /etc/fstab file or fail to specify what partition is to be mounted as the root filesystem, you will be stuck within a temporary, read-only filesystem created by the kernel, located only in RAM. It uses this to switch into the supposed root filesystem. You do not want to be stuck here.

So we create a base /etc/fstab:

```
cat > /etc/fstab << "EOF"
# Begin /etc/fstab

# file system  mount-point  type     options             dump  fsck
#                                                              order

/dev/<xxx>     /            <fff>    defaults            1     1
/dev/<yyy>     swap         swap     pri=1               0     0

# End /etc/fstab
EOF
```

Modify the first entry with the partition location of this system, and the type of filesystem. The second entry should contain the location of your swap partition. If you don't intend on using one, remove the line.

### The End

Congratulations! You've finished building your Debian system, and now have the complete range of functionality expected of a Debian system on top of your regular LFS install.

### Comments, suggestions, bugs

If you found this guide useful, or have suggestions, the author would love to hear from you. Email him at 

scottwilliambeasley AT gmail .com . Replace the AT with @, and remove spaces.

Also, contributions or additions to this manual are very much welcomed. You may do so by using this repository:

https://github.com/scottwilliambeasley/debian-from-scratch/
