---
title: "Bash: Ignore error of particular command" 
tags: 
    - bash
    - linux
categories:
    - how-to
description: "some_command || true"
---
Add `|| true` after the command to ignore the error and continue running the script.

```shell
with_error() {
  false
}

set -o pipefail
set -e

with_error() || true
echo "After with_error() || true" # Prints it in terminal 
with_error()
echo "After with_error()"         # Does not print it in terminal 
```
