---
title: "Git: Recent branches" 
tags: 
    - git
    - linux
description: "git reflog show --pretty=format:'%gs ~ %gd' --date=relative | grep 'checkout:' | grep -oE '[^ ]+ ~ .*' | awk -F~ '!seen[$1]++' | head -n 10 | awk -F' ~ HEAD@{' '{printf(\"  %s\\n\", $1)}''"
---
Add the snippet as alias into `~/.gitconfig`:
```shell
[alias]
    lb = !git reflog show --pretty=format:'%gs ~ %gd' --date=relative | grep 'checkout:' | grep -oE '[^ ]+ ~ .*' | awk -F~ '!seen[$1]++' | head -n 10 | awk -F' ~ HEAD@{' '{printf(\"  %s\\n\", $1)}'
```

Inspired by [Scott Stafford](https://ses4j.github.io/2020/04/01/git-alias-recent-branches/)
