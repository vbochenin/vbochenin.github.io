---
title: "Docker: clean up " 
tags: 
    - docker
categories:
    - how-to
description: docker system prune -f --filter "until=24h"
---

Use the snippet to clean up old and stale containers and images

```shell
docker system prune -f --filter "until=24h" [--volume]
                    |    |---  older than 24 h 
                    |---------   do not ask confirmation
```

[Docker CLI reference](https://docs.docker.com/reference/cli/docker/system/prune/)