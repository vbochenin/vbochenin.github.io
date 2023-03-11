---
title: "Git: Change file permissions" 
tags: 
    - git
description: "git update-index --chmod=+x script.sh"
---

1. `git ls-tree HEAD` get access rights for the script
2. `git update-index --chmod=+x script.sh`  change permissions rights