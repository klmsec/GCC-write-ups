# GCC Write-Ups

## 1. Description

This repository is built to contain all the write-ups/posts from the [gcc-ensibs.fr](https://gcc-ensibs.fr) website. It is made in such way so that when a new post is added to the `main` branch, it is automatically pulled into [GCC-ENSIBS/gcc-ensibs.fr](https://github.com/GCC-ENSIBS/gcc-ensibs.fr).

## 2. Write your own post

First, create a branch named with your post and switch to it. 

###  2.1 File structure 

Once you've created your branch, create a directory at the root of the repository (preferably with a name linked to your post title). Then, `cd` into your newly created directory. The structure should be the following: 
```
.
└── <name_of_your_post>
    ├── assets
    │   ├── cover.png   -> the cover image used for the post
    │   └── *           -> all your other assets go into this directory
    ├── index.fr.md     -> your blogpost written in markdown, french version (optional)
    └── index.md        -> your blogpost written in markdown, english version (mandatory)
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

Currently, only two languages are available on the blog, english (main) and french (secondary). If you wish to write your post in french too, please add a `index.fr.md` in your post directory. This isn't mandatory, if you choose not to write any french translation, your post will be available in english for people who have set the website language to french.

## 3. Have your post published

Once you're satisfied with your post, push the changes to your branch. 

Please open a pull request merging your addition into the main branch. Once merged by a staff member of GCC, your post will be automatically published within a few seconds on `gcc-ensibs.fr`