---
title: "Docker: Setup to run without sudo" 
tags: 
    - docker
    - linux
categories:
    - how-to
description: "sudo usermod -aG docker $USER"
---
Use the snippet to add current user into `docker` group:
```shell
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```
