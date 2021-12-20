---
title: "CI/CD with Github Actions and Argo CD"
date: 2021-12-19
---

Now that we've built and deployed our site. Lets setup some simple CI/CD to help speed up development and content creation (i.e. blog posts).

Since this is a small personal project with minimal dependencies, Github Actions is a great choice and takes advantage of free compute. The only job we need to setup is building and push our docker image.

A simple buildAndPush job that triggers off our main and develop branches. Also includes the option to manually trigger for other branches if needed.

{{< highlight yaml >}}
name: buildAndPush

on:
  push:
    branches:
      - develop
      - main
    paths-ignore:
      - '**/readme.md'

  # allows manual dispatch
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    ...
{{< /highlight >}}

Our job needs to:
- generate build
- provision docker (buildx, login to registry)
- build image
- push image

Github Actions ships with pre-defined "actions" as well as a vast community marketplace. There's a few tasks 

From Argo's site:
>What is [Argo CD](https://argo-cd.readthedocs.io/en/stable/)?  
>  
>Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

