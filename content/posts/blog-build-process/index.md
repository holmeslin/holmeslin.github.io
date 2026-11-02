---
title: "本站建置過程紀錄"
date: 2020-07-07T14:29:40+08:00
categories: [devops]
tags: [hugo, github-pages]
draft: false
description:
aliases:
  - /post/2020/07/blog-build-process/
---
> 紀錄建置過程

> 註：本文記錄的是 2020 年的舊架構（hugo-theme-even + peaceiris/actions-gh-pages），本站目前已改用 toha 主題與 GitHub Pages 官方部署流程。

<!--more-->

## Setup 1

- 在 GitHub 創立 `holmeslin.github.io` Repository
- 創立 `develop` 分支
- 把預設分支設為 `develop`
- clone repository
- 創立 Hugo Site
- 創立 `Action` , 貼入以下內容

    ```yaml {linenos=table,linenostart=1}
    name: Github-Pages
    on:
      push:
        branches:
          - develop

    jobs:
      deploy:
        runs-on: ubuntu-18.04
        steps:
          - uses: actions/checkout@v1
            with:
              submodules: true

          - name: Setup Hugo
            uses: peaceiris/actions-hugo@v2
            with:
              hugo-version: '0.57.2'
              extended: true

          - name: Build
            run: hugo --gc --minify --cleanDestinationDir

          - name: Deploy
            uses: peaceiris/actions-gh-pages@v3
            with:
              github_token: ${{ secrets.GITHUB_TOKEN }}
              publish_branch: master
              force_orphan: true
    ```

- 下載 Hugo Theme 調整內容
- push 上去 `Action` 就會編譯之後合併至 `master`
