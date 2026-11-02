---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
lastmod: {{ .Date }}
draft: true
description: ""
tags: []
categories: []
menu:
  sidebar:
    name: "{{ replace .File.ContentBaseName "-" " " | title }}"
    identifier: {{ .File.ContentBaseName }}
    parent: ""  # one of the sidebar group identifiers in hugo.yaml: go, javascript-web, nginx-server, clean-code, site
    weight: 10
---

<!--more-->
