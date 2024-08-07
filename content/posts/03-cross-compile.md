+++
title = 'Accelerating Docker Builds with Cross Compilation on GitHub Runners'
date = 2024-07-08T18:23:00+01:00
+++


## Motivation
Building Docker images for multiple architectures on GitHub runners, which currently support only the amd64 architecture, can be a time-consuming process. Utilizing QEMU for emulation allows for multi-architecture builds, but it significantly increases build times. In this post, we will explore how cross-compilation can dramatically decrease Docker build times, based on an example from the [Kuadrant Limitador project](https://github.com/Kuadrant/limitador/pull/222) where I implemented these changes.

## Introduction
Docker multi-architecture images are essential for ensuring that applications run smoothly on various hardware platforms, from x86_64 servers to ARM-based devices. However, building these images efficiently is a challenge, particularly when working within the constraints of GitHub runners without using your own self-hosted runner. By leveraging cross-compilation, developers can bypass the slow emulation process provided by QEMU, leading to faster builds and more efficient CI/CD pipelines.

## The Problem
GitHub free runners effectively support only the amd64 architecture, which complicates the process of building Docker images for other architectures, such as ARM. While GitHub provides one free ARM64 runner (`macOS-latest`), it [does not support nested virtualization](https://github.com/orgs/community/discussions/69211#discussioncomment-7197681) which is required for building Docker images. Consequently, using QEMU for emulation allows for the creation of multi-architecture images, but it significantly increases build times. 

This delay is detrimental to development workflows, causing longer feedback loops and slower deployment cycles. Although GitHub now offers ARM runners on their enterprise version, this option wasn't available at the time of our pull request. GitHub plans to offer free ARM runners later in the year, but for now, we need a different solution.

## Case Study: Kuadrant Limitador
The [Kuadrant Limitador project](https://github.com/Kuadrant/limitador/pull/222) faced this exact challenge. Initially, the project relied on QEMU for building ARM images, which resulted in long build times. By adopting cross-compilation, we managed to substantially reduce the build duration, enhancing our development process.

### Before Cross Compilation
Using QEMU, the build process for ARM images was slow and resource-intensive. The builds often took considerably longer than those for amd64, delaying the overall CI/CD pipeline.

### After Cross Compilation
We introduced cross-compilation into our build process. By setting up the build environment to compile for ARM on an amd64 machine, we effectively bypassed the need for QEMU emulation. This change led to a significant decrease in build times, as demonstrated by our [pull request](https://github.com/Kuadrant/limitador/pull/222).

To further optimize the build process, we distributed the builds across multiple GitHub runners. This parallelization allowed us to reduce the total build time even further, leading to a much more efficient workflow.

### Build Time Comparison
Here's a comparison of our build times before and after implementing cross-compilation and parallel builds:

![build-time-before-after](/03-build-times.png)

## Implementing Cross Compilation

### Prerequisites
Ensure you have the following tools installed:
- Docker
- Go (if building Go applications)
- Buildx (a Docker CLI plugin for extended build capabilities with BuildKit)

### Setting Up Buildx
First, enable Buildx:
```sh
docker buildx create --use
docker buildx inspect --bootstrap
```

### Creating a Multi-Platform Builder
Create a builder instance that supports multiple platforms:

```sh
docker buildx create --name mybuilder --use
docker buildx inspect mybuilder --bootstrap
```

### Cross-Compiling the Docker Image
Modify your Dockerfile to support cross-compilation. For a Go application, this could look like:
```Dockerfile
FROM --platform=$BUILDPLATFORM golang:1.18 as builder
WORKDIR /app
COPY . .

ARG TARGETOS
ARG TARGETARCH
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o myapp .

FROM --platform=$TARGETPLATFORM alpine:latest
WORKDIR /app
COPY --from=builder /app/myapp .
ENTRYPOINT ["./myapp"]
```

### Building and Pushing Multi-Architecture Images
Use the following command to build and push the images for multiple architectures:

```sh
docker buildx build --platform linux/amd64,linux/arm64 -t yourusername/yourimage:tag --push .
```

### Example Workflow Configuration
For a complete example of using distributed runners for parallel Docker image builds, refer to Docker's official documentation [here](https://docs.docker.com/build/ci/github-actions/multi-platform/#distribute-build-across-multiple-runners).


## Conclusion
Cross-compilation is a powerful technique for optimizing Docker builds, particularly when dealing with multiple architectures. By leveraging cross-compilation and distributing the build process across multiple runners, the Kuadrant Limitador project significantly reduced its build times, illustrating the potential for enhanced efficiency in CI/CD pipelines. Implementing these strategies in your projects can lead to faster builds, quicker feedback loops, and ultimately a more productive development workflow.

While I implemented the changes late last year in November, cross-compilation is still a good route to decreasing build times. By understanding and applying these strategies, developers can overcome the limitations of GitHub runners and build robust, multi-architecture Docker images efficiently. Even though GitHub plans to offer [free ARM runners later in the year](https://github.blog/2024-06-03-arm64-on-github-actions-powering-faster-more-efficient-build-systems/), cross-compilation remains a valuable approach to streamline the development process and improve performance today.
