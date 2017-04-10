---
layout: post
title: Easily deploying test environments from CI builds with Travis, AWS and Slack
author: tatut
excerpt: Easy setup of test environments is important for feature branch based development. This post shows how to automatically deploy a new test environment with a simple chat interface.
tags:
- CI
- AWS
- cloud
- slack
---

Our project uses a git flow process: a new branch is created for each new feature. The feature is
developed in its entirety in the branch. This includes tests, database migrations and the code
itself. When the feature is ready, a GitHub pull request is made and another developer tests and
reviews the code.

We wanted to make testing another branch easy for developers. Developers will most likely have
their own branch in progress on their local machine and we wanted a system where developers
don't need to hop between their own branch and another one. Initially we made six development
test machines where the initial developer would deploy a new version for testing but this was
cumbersome as it needed manual deployment work and a reservation system to reserve a test
machine for your branch.

## Along comes AWS

We are using the excellent [Travis CI](https://travis-ci.org/) for building our project and
running the automated test suites. We had the idea that we could automatically push the build
artifacts to S3 so that the latest build of a branch (along with a database dump) is always
available for deployment. Luckily Travis has direct support for S3 so the configuration is
a simple snippet in `.travis.yml` file:

```yml
deploy:
  provider: s3
  region: eu-central-1
  skip_cleanup: true
  bucket: "harjatravis"
  acl: public_read
  local_dir: s3-deploy
  on:
    all_branches: true
```

This instructs Travis to push all branches to our bucket. We set the bucket configuration
to automatically expire the artifacts after 2 weeks so that we don't accumulate old builds
for too long.