---
title: "Locking yourself out of SDDM with .bashrc"
date: "2026-03-01"
tags:
  - linux
  - rice
categories:
  - linux
  - til
---
# Transferring 

I'm going to [SCaLEx23](https://www.socallinuxexpo.org/scale/23x). I don't tend to use my laptop as a primary device so I was transferring *everything*.   
  
I literally can't live without my aliases and shortcuts so in my rush I somehow moved a copy of my `.bashrc` into `~/.bashrc.d`. Which doesn't sound like a big deal except I have this in my main file.  
  
  ```bash    
  if [ -d ~/.bashrc.d ]; then
  for rc in ~/.bashrc.d/*.sh; do
    if [ -f "$rc" ]; then
      . "$rc"
    fi
  done
  fi 
  ```
  
 Why load everything in? Because I'm lazy.   
   
 I find out on next login that I'm stuck in a login-loop. Getting in through TTY3 check the thousands of SDDM error crashing messages in `journalctl` and remove the rogue file. 
  
 # The Fix
   
I've since moved everything to only loop over items ending in `.sh` to avoid this unwanted files being sourced.
 
