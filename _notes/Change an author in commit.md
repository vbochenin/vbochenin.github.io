---
title: "Git: Change author of pushed commits" 
tags: 
    - git
description: 'git commit --amend --author="new-author-name new-author@email.com" --no-edit'
---
1. Switch to rebase mode
   ```bash
   git rebase -i HEAD~<N>
   git rebase -i master
   ```
3. Mark required commits to edit (e)
4. Repeat required times
   ```bash
   git commit --amend --author="<new author name> <new-author@email.com>" --no-edit
   git rebase --continue
   ```
5. Check if everything is ok
   ```bash
   git log
   ``` 
7. Push changes
   ```bash
   git push --force
   ```
