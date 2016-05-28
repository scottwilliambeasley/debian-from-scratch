# debian-from-scratch

An instruction manual for teaching Linux From Scratch users how to make a custom Debian-powered distro.

##Why Debian from Scratch?

The original Linux from Scratch manual is purposefully vague as to what technique one should use to manage software dependencies. The suggestions that it gives, while no doubt being interesting exercises in package management, are not necessarily heavy-duty answers to a system administrator who intends on managing his time efficiently. 

The disadvantage of compiling everything to create a fully-fledged system is time. After one builds an LFS system for the first time, he/she is apt to realize that managing dependencies can be an arduous task, to say the least. Going through the insufferable exercise of hunting down dozens to possibly hundreds packages, mapping dependencies, configuring and installing these dependencies in the correct order, just to install a single piece of software, is not a viable alternative to a system administrator who values his time.

The answer to this problem, is obviously to use a package manager. There are many available package managers, the most popular of which are the Debian-based package management suites (dpkg and apt), and the Red Hat based package management suites (rpm and yum). 

This manual will teach you how to build your system utilizing the Debian set of package management tools by utilizing the temporary system environment created in Linux From Scratch.

## Purpose for this project

I chose to make this manual because I have seen woefully old guides on the internet teaching others how to get dpkg and apt running on their own custom linux, and people asking on various forums on how to install dpkg and apt, but not getting the help that they need. These guides are outdated and no longer contain up-to-date information, which I intend to fix here in this manual.

This project intends to be a community resource to help those interested in creating their own custom system from the ground up, while fully taking advantage of the Debian suite of package management, dpkg and apt, in order to solve the problems of package dependency installation and management.

## How to use this manual? 

This manual is designed to be used after completing all instructions up to the end of Chapter 5 of the Linux From Scratch book, version 7.9. One first follows the instructions of the original LFS book, and builds the temporary system that is created in LFS Chapter 5. One is required to have a fully-functional temporary system which the the outcome of Chapter 5. 

After completing the preliminary preparations, one then consults this manual and follows it step-by-step.

Like the original LFS manual, when dealing with packages to be compiled, each section already assumes that you have extracted the source code and have changed your main directory into the main folder of the extracted content. However, when dealing with .deb files no such extraction is needed. One only needs to follow the instructions while having the .deb file in your current directory.

## Overview of our Method for Building a Custom Debian System

In the original Linux From Scratch book, we created a cross-toolchain, using the toolchain native to our system. We then used this cross-toolchain to create a native toolchain, which ended up being the temporary '/tools' system environment. This was Chapter 5's goal. We then used this temporary system to build our final system, which was Chapter 6's goal.

In Debian From Scratch, we branch off from the end of Chapter 5. Instead of using the toolchain and other utilies installed in `/tools` to compile every single part of the final system, we instead extend this toolset to include the minimum required dependencies for installation of Debian's package manager, dpkg.

We then compile and install dpkg as the first part of our final system, and use dpkg along with some clever dependency hacking to satisfy all remaining dependencies needed to install apt. This allows us to rely on apt for the overwhelming majority of tasks involving the installation of software onto our new system, and to allow us to avoid the exercise in tedium that is manual dependency management.

##Preparing the virtual kernel filesystem mount points
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

#####Creating temporary links linking to /tools/bin/bash
In order the for pre and post-installation scripts that come inside a standard .deb file to work, these need to be able to have access to a shell. These typically either specify using `/bin/sh` or `/bin/bash`. Without access to a shell at the exact location that the script specifies, the installation script will fail thus causing the installation of the package itself to fail.

We must mitigate this problem by making sure that the `/bin` directory already exists, and then creating symlinks from these two locations towards the bash we have in our temporary `/tools` environment. 

When we get to the point of installing debian's "priority:essential" packages, which include both of these shells, these symlinks will be overwritten with native copies of these binaries.

```
mkdir /bin
ln /tools/bin/bash /bin/bash
ln /tools/bin/bash /bin/sh
```

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


###Installing apt

Before we can install `apt`, and use this to automatically install the most of the rest of our system software, we have to install its immediate dependencies on our target system first.

![](https://cdn.rawgit.com/scottwilliambeasley/debian-from-scratch/master/images/apt-dependencies.svg)
#####Figure 2 - The apt dependency tree, one level deep

Each of these immediate dependencies has their own set of dependencies to fulfill. We shall start by completing the dependency tree for `debian-archive-keyring`. Unlike the compilation process needed to install `dpkg`, the process we now use to install software is by installing .`deb` files using `dpkg`. 

####Installing debian-archive-keyring

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

`dpkg -i (location of libgcc1)`

### Creating configuration files

Before we proceed with updating `apt's` cache, let's create some much-needed configuration files in order to do so, as well as change how `bash` handles special keyboard presses. 

####/etc/resolv.conf

`/etc/resolv.conf` is a file needed in order for your system to have DNS resolution. Without this file, no resolution typically happens, which makes updating `apt`'s cache of installable packages via `/etc/apt/sources.list` difficult.

```
cat > /etc/resolv.conf << "EOF"
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF
```

####/etc/apt/sources.list

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

####/etc/hosts

`/etc/hosts` is a file used to contain mapping of IP addresses to hostnames. This file is usually checked before DNS queries, at the very least, this should contain your `ipv4` and `ipv6` loopback addresses.

```
cat > /etc/hosts << "EOF"
127.0.0.1       localhost

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
EOF
```

####/etc/hostname

`/etc/hostname` is used to contain your host's DNS name. Edit this if you like.

```
cat > /etc/hostname << "EOF"
debianfromscratch
EOF
```

####/etc/inputrc

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

###Updating apt's package lists

We are now ready to update our list of packages and take full advantage of `apt`. To do this, we update our local keyring of valid Debian developer gnu pgp signatures using `apt-key update`, and then update with `apt-get update`.
```
apt-key update
apt-get update
```

###Creating user and group databases

Before we install more software, we must make sure that our password, group and authentication mechanisms are all in place. This is because some packages will require the adding of a new user or group to the system as part of their installation process. Without these base functionalities already in place, installation of said packages will fail.

#####debianutils (
We install debianutils to provide the `tempfile` command needed by one of `base-passwd`'s installation scripts. Without this command, installation of `base-passwd` will fail.
`apt-get install debianutils`

#####base-passwd 
We install `base-passwd` to provide standard the standard minimal `/etc/passwd` and `/etc/group` files, which are the same across all debian systems. It does this by running the `update-passwd` binary upon its installation.
`apt-get install base-passwd`

#####Creating /etc/shadow and /etc/gshadow
We have to manually create `/etc/shadow` and `/etc/gshadow`, as the `passwd` package will fail to configure if it cannot find these files:

`touch /etc/shadow /etc/gshadow`

#####login 
We then install the `login` package, which gives us the ability to establish new sessions on the system with `login`, privilege escalation with `su`, the linux pluggable authentication module (PAM) files for both said binaries, a fake shell `/bin/nologin`,  and the `/etc/login.defs` file which is essential for group creation. There are more functionalities included with this package, but these are the most mentionable.
`apt-get install login`
 
#####passwd 
We then install `passwd` package, which provides the lion's share of utilities and configuration files used to create and manipulate user and group account information.
`apt-get install passwd`


#####adduser 
We must also install the `adduser` package, because this provides us with the default `/etc/adduser.conf` file which will be needed to install new users properly.
`apt-get install adduser`

#####Establishing root password and shadowfile entries
With all the aforementioned utilities and packages installed, our system is now capable of the full functionality of user & group account manipulation.

At this point, we should run passwd to change our root password. 
`passwd root`

We should then run `pwconv` to convert our /etc/passwd entries into shadow entries in `/etc/shadow`
`pwconv`a
