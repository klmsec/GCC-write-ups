# GCC Write-Ups

## 1. Description

This repository is built to contain all the write-ups/posts from the [gcc-ensibs.fr](https://gcc-ensibs.fr) website. It is made in such way so that when a new post is added to the `main` branch, it is automatically pulled into [GCC-ENSIBS/gcc-ensibs.fr](https://github.com/GCC-ENSIBS/gcc-ensibs.fr).

## 2. Write your own post

    1. Fork this repository
    2. `git clone https://github.com/<your_username>/write-ups`
    3. `cd write-ups`
    4. git checkout -b <your_branch_name>
    5. add your post by following 2.1, 2.2, 2.3
    6. `git add .` 
    7. `git commit -m "add new post"`
    8. git push origin <your_branch_name>
    9. Create a pull request from your repo to our main branch

###  2.1 File structure 

Once you've created your branch, create a directory at the root of the repository (preferably with a name linked to your post title). Then, `cd` into your newly created directory. The structure should be the following: 
```
.
└── <name_of_your_post>
    ├── assets
    │   ├── cover.png   -> the cover image used for the post
    │   └── *           -> all your other assets go into this directory
    ├── index.fr.md     -> your blogpost written in markdown, french version 
    └── index.md        -> your blogpost written in markdown, english version
```

### 2.2 Markdown files structure

The head of your markdown index file should always be the following:
```
---
author: <author_name>
title: <post_title>
description: <short_description>
slug: <slug-link>                   
date: 2023-12-31 00:00:00+0000
image: assets/cover.png
categories:
    - <Cat1>
    - <Cat2>
    - ...
tags:
    - <Tag1>
    - <Tag2>
    - ...
---

# Title

// Rest of your blogpost
```

> The slug will be the URL used to access the post, for example if the slug is im-pretty, the full link to the post will be gcc-ensibs.fr/p/im-pretty, so make sure it is URL friendly and doesn't overlap an other already existing post :).

###  2.3 English language and translations

Currently, only two languages are available on the blog, english (main) and french (secondary). If you wish to write your post in french too, please add a `index.fr.md` in your post directory. Even if you don't wish to write a french version, please copy `index.md` to `index.fr.md`. This will allow readers who have french language selected to still be able to view your content but in english :). 

## 3. Have your post published

Once you're satisfied with your post, please open a pull request merging your addition into the main branch. Once merged by a staff member of GCC, your post will be automatically published within a few seconds on `gcc-ensibs.fr`
