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

Turns this 
```bash
figsystems on main 
  11:48 ❯ /usr/bin/ls
archetypes  content  hugo.yaml  README.md  themes
```
to this
```bash
figsystems on main 
  11:48 ❯ ls
Permissions Size Date Modified Git Git Repo Name
drwxr-xr-x@ 1.1k  5 Feb 14:18   -- - -       archetypes
drwxr-xr-x@  70k  5 Feb 14:18   -- - -       content
.rw-r--r--@ 3.4k  5 Feb 15:57   -- - -       hugo.yaml
.rw-r--r--@  237  5 Feb 14:18   -- - -      󰂺 README.md
drwxr-xr-x@    0  5 Feb 14:18   -- - -       themes
```
