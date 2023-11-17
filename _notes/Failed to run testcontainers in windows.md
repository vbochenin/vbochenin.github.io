---
title: "Failed to run testcontainers in windows" 
tags: 
    - docker
    - testcontainers
categories:
    - how-to
description: "bundle exec jekyll serve --livereload"
---
**Description:** failed to map docksock file with  docker on windows

**Solution:** 
> If you are using Docker Desktop, you need to configure the TESTCONTAINERS_HOST_OVERRIDE environment variable to use the special DNS name host.docker.internal for accessing the host from within a container, which is provided by Docker Desktop: -e TESTCONTAINERS_HOST_OVERRIDE=host.docker.internal
[Patterns for running tests inside a Docker container](https://java.testcontainers.org/supported_docker_environment/continuous_integration/dind_patterns/)
