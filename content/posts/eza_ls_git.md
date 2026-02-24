---
title: "Rice"
description: "Making Linux Cool - Dynamically"
date: "2026-02-24"
tags:
  - rice
  - bash
categories:
  - linux
---
I like the fun linux commands that rice out your setup but I don't like having to remember when/how to use them. 

I've got my `.bashrc` that calls everything in `.bashrc.d/*`. In there I've started throwing in files with custom config options. 

Here's a quick ls function that's context aware. I'll be expanding it as I learn better ways to use eza or similar listing tools.

```bash
alias ls='_ls' 
# ls: use eza with git info inside git repos, plain ls elsewhere
_ls() {
    if git rev-parse --is-inside-work-tree &>/dev/null; then
        eza -l -h --git --git-repos --total-size --no-user --git-ignore --icons "$@"
    else
        command ls "$@"
    fi
}

```
