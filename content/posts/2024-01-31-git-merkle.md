---
title: "new project: git-merkle (gm)"
date: 2024-01-31
last-update: 2024-01-31
draft: false
tags: ["git", "programming", "organization", "merkle-tree", "art"]
categories: ["programming" ]
---

GM! 
![](/photos/2024-01-31-merkle.md/gm-tree.jpeg)

Just a short note describing a recent new project of mine, [git merkle (gm)](https://github.com/thor314/gm)!

`gm` is a recursive tree of all the repos I've worked on since 2017 after a little spring cleaning; I removed about 40% of my pre-exisiting github repo history in the process of writing `gm`. There are about 90 repos contained in `gm`, and counting. I removed about 60. 

## Why did you do this?!
This is a very real question. Organizing and cleaning up about 6 years of git history, plus writing a [script](https://github.com/thor314/.cron/blob/main/git_merkle.fish) to keep them organized going forward, took about 6 hours across two days. Submodules are somewhat infamously finicky, and the last thing I want to be doing is managing merge conflicts on a nested tree of my commit history.

That is, if a commit is made to a submodule at depth $d$ in the tree, say in my repo `helix`, for which the path is `gm/linux/.files/helix`, then $d$ new commits are added to the git-merkle tree.[^1]

I have a few reasons. One is that it's very nice to have a copy of the full history of my work on hand to [ripgrep](https://github.com/BurntSushi/ripgrep) through. Another is that it's a nice way to keep my work presentable for others. But most especially, this is one representation of all the work I've done for the last 6 years, and I'm proud of it. This repo is a sort of monument to the pride I feel in the all the work I've done. As a cryptography engineer, I find it apropos to represent that work as the workhorse data structure of blockchain cryptography, the merkle tree.

![](/photos/2024-01-31-merkle.md/git-merkle-tree-retro.jpeg)
GM.

```
root = h(n1, n2)
|               \
n1=h(n3,n4)      n2=h(n5,n6)
|         |      |         |
n3        n4     n5        n6
```
*a merkle tree where the leaves are where data is stored and the root is a single hash*

[^1]: For those of you who are rusty on cryptographic data structures 101, a merkle tree is a tree data structure where the leaves contain the data, and every intermediate node is the hash of it's leaves, see above. It's useful for committing to a bunch of data in a single hash (256 bytes), that cryptographically represents the compressed state of the entire set of data, even if there's a ton of data. Each git commit is produces the hash of the state of the repo, so the submodule tree is a just a tree of commit hashes.