---
title: "@Rollback annotation in test does not rollback anything" 
tags: 
    - spring
    - unittests
categories:
    - how-to
description: "use @Transactional annotation in test"
---
**Description:** Test with `@Rollback` annotation creates some records, but this records stay in DB after the test method’s exit.

**Solution:** Mark the test with `@Transactional` annotation to rollback the test as complete transaction
