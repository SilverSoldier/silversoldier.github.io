---
layout: post
title:  "Pathogen, Vim, Windows"
description: An guide to using pathogen for installing vim plugins on Windows.
img:
date: 2018-06-13  +1120
---

# Using Pathogen with vim on Windows

Wanted to install plugins on vim on Windows. There are 2 options - native and Cygwin.

- Native: Using cmd and powershell
- Cygwin: Using cygwin (duh!)

I tried both installation methods. Did not get it to work with cygwin.

For native installation there were varying options like using .vim or vimfiles for the autoload and bundle folders. There was confusion on whether to put it in `C:\Users\<username>` or `C:\Program Files\<path to vim>`.

These are the final set of steps.

1. Downloading vim
+ Use [chocolatey](https://chocolatey.org)
+ Just `choco install vim`.
+ It should get installed as `C:\Program Files (x86)\vim\vim80`. (Where vim80 is version and will vary as necessary).

2. Location of .vimrc
+ Can either have local (per user) settings or global settings. Local settings ain't gonna hurt anybody so,
+ Edit `C:\Users\username\.vimrc` as necessary.

3. Using pathogen
+ **Pathogen Location**:
  + I guess there are local and global settings for this as well. Technically, there should be. Practically, good luck trying to find out how to do so. I've used global settings. Global settings ain't gonna hurt anybody as long as it is your system, so,
  + Use the curl instruction (maybe with cygwin) and put pathogen.vim in `C:\Program Files (x86)\vim\vim80\autoload\`.
  
4. Using your plugin of choice
+ `git clone <plugin> C:\Program Files (x86)\vim\vim80\plugin\`
