---
published: true
layout: post
date: 2019-07-28T00:00:00.000Z
excerpt_separator: <!--more-->
tags:
  - docker
  - gitlab
  - cicd
title: >-
  Faster CI builds when using Docker-in-Docker on GitLab
---

![Docker-in-Docker (DIND)](https://i.imgur.com/zFBWGQR.png)

Docker-in-Docker (DinD) is the [recommended way to build and test Docker images in GitLab](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker-executor). 

By running each build with it's own fresh Docker daemon, you get the benefit of having a clean build environment and the ability to run concurrent jobs without any conflicts from other build containers.

However, your build times suffer because you can no longer cache the image layers on disk between builds, instead having to download them. 

This post will tell you how to get a speed increase by adding a couple of lines to your `.gitlab-ci.yml` file.

<!--more-->

_**TL;DR:** Pass `--max-concurrent-downloads` with a value greater than the default of 3 to your DinD service's entrypoint to increase the speed of your layer downloads when using `docker pull` & `docker build --cache-from` in your CI pipelines._

## Example DinD Setup

While you can't cache the image layers on disk between builds like you would if you reused the same Docker daemon, you can [specify a pre-existing image to use as a cache](https://medium.com/@gajus/making-docker-in-docker-builds-x2-faster-using-docker-cache-from-option-c01febd8ef84) during `docker build` using `--cache-from`. As you don't have access to the image on disk, you need to download it first via `docker pull`. For example:

```yaml
image: docker:stable

services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  
stages:
  - build
  - test

before_script:
  - docker login -u ${CI_REGISTRY_USER} -p ${CI_REGISTRY_PASSWORD} ${CI_REGISTRY}

build:
  stage: build
  script:
    # Try to pull a previously built image, if it fails ignore the failure
    - docker pull ${CI_REGISTRY_IMAGE}:${CI_BRANCH_SLUG} || true
    # Build our new image using the previous one as a cache
    - docker build --cache-from ${CI_REGISTRY_IMAGE}:${CI_BRANCH_SLUG} --tag ${CI_REGISTRY_IMAGE}:${CI_PIPELINE_ID} .
    # Push image to use in rest of pipeline
    - docker push ${CI_REGISTRY_IMAGE}:${CI_PIPELINE_ID}

test:
  stage: test
  script:
    # Pull image built in build stage
    - docker pull ${CI_REGISTRY_IMAGE}:${CI_PIPELINE_ID}
    # Run tests
    - docker run ${CI_REGISTRY_IMAGE}:${CI_PIPELINE_ID} test
    
# Excluded: Pushing image to GitLab Registry as ${CI_REGISTRY_IMAGE}:${CI_BRANCH_SLUG} and pushing to remote registry / deploying
```

In the build stage, we pull the image previously built and released on the branch and then re-build the container using that image as the cache. If only the code has changed, then only 1 layer will be different (a simple COPY or ADD) meaning that your docker build will be extremely quick! We then push a new image to be used in the pipeline later.

Then in later test stages (or any other stages), we pull the image we built and run tests etc.

## Downloading Image Layers

The bottleneck may now become your network. Docker images are broken down into layers, and by default the Docker daemon downloads 3 layers at a time.

```
b234f539f7a1: Pulling fs layer
55172d420b43: Pulling fs layer
5ba5bbeb6b91: Pulling fs layer
d0fa7cfafe64: Pulling fs layer
9ed2dcf5d0d2: Pulling fs layer
e8f13a9aba80: Pulling fs layer
d0fa7cfafe64: Waiting
9ed2dcf5d0d2: Waiting
e8f13a9aba80: Waiting
```

This is a sensible value to suit the limits of most connections, but often unsuitable when you are running powerful CI build runners with fast network connections.

To get more juice out of your setup, it'd be nicer if a larger number of layers could be downloaded concurrently instead of waiting around.

## max-concurrent-downloads

Enter `--max-concurrent-downloads`. This is an [option you can pass your Docker daemon](https://docs.docker.com/engine/reference/commandline/dockerd/), (there's also `--max-concurrent-uploads`).

You can pass it to the `docker:dind` service by altering your `.gitlab-ci.yml` definition accordingly:

```yaml
services: 
  - name: docker:dind
    entrypoint: ["dockerd-entrypoint.sh"]
    command: ["--max-concurrent-downloads", "6"]
```

By adding the flag to the `command`, [it will pass them as command line options to the `docker:dind` container](https://github.com/docker-library/docker/blob/306841553919fba3de31766c2ba44b9e2cc5e2ec/19.03/dind/dockerd-entrypoint.sh#L116).

You'll need to tweak and monitor your own setup to find the right value of concurrent downloads. 

## Further Improvements

To further improve the speed of your image layer downloads, you could consider running a [Docker registry mirror](https://docs.docker.com/registry/recipes/mirror/). 

If you use any public Docker images this will vastly speed up download times if the mirror is on the same LAN as your build runners. Similarly, if your build runners aren't on the same LAN as your GitLab instance this will have the same effect.

Even if your build runners are on the same LAN as your GitLab instance it can help to lessen the load on your GitLab server.

[You can use the same technique of passing command line options to the `docker:dind` service to specify a registry mirror.](https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon)

If anyone else has discovered techniques of speeding up their Docker CI builds especially when using DinD I would love to hear about them! Either comment on this post or tweet me [@JalamoJ](https://twitter.com/JalamoJ)
