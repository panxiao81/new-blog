---
title: "When We Are Saying CentOS is Stabler than Others, What Are We Talking About?"
date: 2022-03-24T01:58:23+08:00
draft: false
categories: ['Linux']
tags: ['linux']
---

We could always hear that someone says CentOS or RHEL is stabler than other distros in China, like Ubuntu or anything else. And similarly, when we are discussing package managers, some opinions say APT or Pacman is better than YUM. But are these things true? I can’t agree with most of them.

<!--more-->

## System Stability or ABI Stability?

CentOS is stabler than others, someone says that. But when you ask those people why it’s stabler, maybe they will say because the source code of CentOS comes from Red Hat and it has been fully tested by RHEL users.

For these opinions, I should say that’s true. But some system administrators update their Linux kernel for newer functions, and some others even compile their own Glibc and GCC for newer software on old systems like installing docker on CentOS 6.

If you ever tried to build your own Linux From Scratch, you should know that the Linux Kernel, the Glibc in the system, and the GCC complier’s version should be stable, and all the software should also be compiled with that version. When we were doing that, the whole system could be called stable because of the same system’ s ABI.

Linux is not the same as Windows or FreeBSD, the software that users are using should be shipped with the system. That is all because the Linux ABI is not stable. And the whole system is pieced together with a branch of open source software, and most of them are coming from the GNU project.

Because of that, there are so many sysadmins afraid about upgrading the system, for they don’t know if the system will boot or the software will run as usual after the system upgrade. Especially for large version number upgrades. Some people even say the most stable ABI on Linux is win32 ABI with Wine, especially for Linux desktop users.

And also, the Linux distribution’s maintainer must compile the software with the newer version of kernel and Glibc, test it, and shipped the compiled binary with the Linux distributions. And only the whole system with that software that is being fully tested could be called stable.

So, as I said above, if you update your system kernel or Glibc, you have broken the stability of the system, and for now, the system could not be called stable anymore.

## The Real World Problem

But in the real world, most of the time the software packages shipped with the distros are not enough for us, and we must use some third-party software. If you buy some software from vendors, they could have been tested the compatibility with the system you are using. But for open source software? Sorry, the software itself “IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND”

## And Also for Some Others

Similarly, when someone talks about package manager, they will say “APT is better than yum” or “Pacman is the best”. Technically the APT is not good because of the ability to solve the dependency and download speed. But the DNF, which is the next generation of YUM, is better than APT. The real reason that those people say better is that the number of the package in Ubuntu’s (or other Debian-based distros) official repository is larger than CentOS’s. or the software’s version shipped with the distribution is newer.

Someone doesn’t like CentOS Stream, they are actually confused CentOS Stream with Arch-Like rolling distros. CentOS Stream is a rolling distribution, but not so “rolling”, it still has a major version, and the main package in this major version is still the same.

At last, you could say RHEL or any other RHEL-derivative distribution is stable, but I want you to know that when we say stable, what we are really talking about.
