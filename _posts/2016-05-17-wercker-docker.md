---
layout: post
title: Creating Minimal Docker Images for Go App with Wercker
---

For years I've been using CircleCI to do the typical CI things: building a project, running tests, packaging & uploading artifacts, and even deploying. Lately I've noticed a lot of reliability and performance issues with CircleCI and figured it was time to explore the CI landscape.  I used [Wercker](https://www.wercker.com) about a year ago to build some Bitbucket-hosted projects that were written in Java. Wercker performed wonderfully, was (and still is) free, and supports both Github and Bitbucket.

So, I decided to move my [StreamMarker](https://github.com/skidder/streammarker) Golang projects to Wercker. The Wercker documentation includes some *very* simplistic [examples](http://devcenter.wercker.com/docs/languages/golang.html) of how to build Golang projects. These were a good starting point, but left out some vital details:

 * building projects that use Go 1.5+ experimental vendoring
 * building projects that include sub-packages

## Go Experimental Vendoring ##
[Glide](https://github.com/Masterminds/glide) is a Go dependency management tool that's grown in popularity since late 2015. Glide supports Go 1.5 experimental vendoring.Projects using Glide include a `glide.yml` file enumerating their explicit dependencies. The Glide tool includes a `glide update` step that examines the `glide.yml` file and the project source to identify dependencies, and writes a `glide.lock` file describing exactly which dependencies and their version to use when building. Dependencies must be downloaded from their origin on each run of the CI build. Hence, it can make CI a little tricky since the Glide tool must be installed.

I searched the [Wercker Steps Registry](https://app.wercker.com/#explore/steps) for Glide build steps.  There were two Step repositories, but they both reference ancient versions of Glide. I created [my own fork](https://github.com/skidder/step-glide-install) that uses Glide `0.10.0` and published the step to the Wercker step repository. This makes the step available for use in the `wercker.yml` file included in each project I intend to build.

## Building Projects with Sub-Packages ##
Wercker does not place the project source on the `GOPATH` by default.  This complicates building in that you must manipulate the `GOPATH` environment variable or copy the project source to the appropriate location on the `GOPATH`. I went with the copy route.

## My Resulting wercker.yml##
Here's are the contents of the `wercker.yml` file for my (StreamMarker Collector)[https://github.com/skidder/streammarker-collector] component.

```yaml
box: golang
build:
  # The steps that will be executed on build
  steps:
    - skidder/glide-install@1.0.2
  
    - script:
        name: Copy source to GOPATH
        code: |
          rm -rf vendor
          mkdir -p ${GOPATH}/src/github.com/skidder/streammarker-collector
          cp -R . ${GOPATH}/src/github.com/skidder/streammarker-collector

    - script:
        name: build
        code: |
          make static-build

    - script:
        name: test
        code: |
          make test

    # Copy binary to a location that gets passed along to the deploy pipeline
    - script:
        name: copy binary
        code: cp streammarker-collector "$WERCKER_OUTPUT_DIR"
deploy:
  steps:
    - internal/docker-scratch-push:
        username: $DOCKERHUB_USERNAME
        password: $DOCKERHUB_PASSWORD
        tag: latest
        ports: "3000, 3100"
        cmd: ./streammarker-collector
        repository: skidder/streammarker-collector
        registry: https://registry.hub.docker.com

```

In order to build a Golang executable that can run in a minimal Docker container, your compilation step must produce a statically-linked binary. This is handled in the `make static-build` step in the `wercker.yml` file provided above.  The `static-build` target in the `Makefile` looks like:

```Makefile
static-build:
	CGO_ENABLED=0 go build -a -ldflags '-s' -installsuffix cgo -v -o streammarker-collector
```
