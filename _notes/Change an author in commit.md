---
title: "Git: Change author of pushed commits" 
tags: 
    - git
description: 'git commit --amend --author="new-author-name new-author@email.com" --no-edit'
---
1. Switch to rebase mode
   `git rebase -i HEAD~<N>`
   `git rebase -i master`
2. Mark required commits to edit (e)
3. Repeat required times
   ```bash
   git commit --amend --author="<new author name> <new-author@email.com>" --no-edit
   git rebase --continue
   ```
4. Check if everything is ok
   `git log` 
5. Push changes
   `git push --force`
