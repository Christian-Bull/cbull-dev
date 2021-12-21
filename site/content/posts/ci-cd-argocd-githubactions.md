---
title: "CI/CD with Github Actions and Argo CD"
date: 2021-12-19
---

Now that we've built and deployed our site. Lets setup some simple CI/CD to help speed up development and content creation (i.e. blog posts).

## Github Actions

Since this is a small personal project with minimal dependencies, Github Actions is a great choice and takes advantage of free compute. 

First lets setup a simple buildAndPush job that triggers off our main and develop branches. It also includes the option to manually trigger for other branches if needed.

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

First some boilerplate to setup our environment.
{{< highlight yaml >}}
    steps:
      - name: set env vars
        run: |
          echo "SHA=${GITHUB_SHA}" >> $GITHUB_ENV
          echo "GITHUB_REF_NAME=${GITHUB_REF_NAME}" >> $GITHUB_ENV
      - name: Checkout
        uses: actions/checkout@v2
{{< /highlight >}}

Install Hugo and generate our build.
{{< highlight yaml >}}
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.90.1'

      - name: Build
        run: cd site && hugo --minify
{{< /highlight >}}

Setup [buildx](https://docs.docker.com/buildx/working-with-buildx/) for multi-arch images.
{{< highlight yaml >}}
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v1.2.0
        with:
          platforms: all
      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v1.6.0
        with:
          version: latest
{{< /highlight >}}

Lastly, a simple build and push task using the github envs we setup earlier.
{{< highlight yaml >}}
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64,linux/arm/v6
          tags: |
            csbull55/cbull-dev:${{ env.GITHUB_REF_NAME }}-latest
            csbull55/cbull-dev:${{ env.GITHUB_REF_NAME }}-${{ env.SHA }}
          push: true
{{< /highlight >}}

Now whenever we push/merge code to the develop or main branch, this job will build and push a docker image to our remote repository for the respective branch.

## Argo CD

Next we'll use Argo CD to sync new builds with the state on the cluster.

From Argo's site:
>What is [Argo CD](https://argo-cd.readthedocs.io/en/stable/)?  
>  
>Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

Argo CD works by monitoring running applications in the cluster and compares them with the target state you've defined in git. When it detects it's out of sync (i.e. you've pushed updated manifests), Argo automatically syncs the app on the cluster to match that target.

Let's setup a simple Argo CD app and register it. I typically use the UI and grab the outputted yaml spec to throw in git.

{{< highlight yaml >}}
project: default
source:
  repoURL: 'https://github.com/Christian-Bull/cbull-dev-infra.git'
  path: manifests
  targetRevision: main
destination:
  server: 'https://kubernetes.default.svc'
  namespace: cbull-dev
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
{{< /highlight >}}

This sets up our source repo and defines the path and branch to watch for changes on.

Once created, lets ensure it's healthy.

{{< highlight bash >}}
➜  ~ argocd app get cbull-dev
Name:               cbull-dev
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          cbull-dev
URL:                http://localhost:8080/applications/cbull-dev
Repo:               https://github.com/Christian-Bull/cbull-dev-infra.git
Target:             main
Path:               manifests
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to main (3982373)
Health Status:      Healthy

GROUP       KIND        NAMESPACE  NAME       STATUS  HEALTH   HOOK  MESSAGE
            Service     cbull-dev  cbull-dev  Synced  Healthy        service/cbull-dev unchanged
apps        Deployment  cbull-dev  cbull-dev  Synced  Healthy        deployment.apps/cbull-dev configured
extensions  Ingress     cbull-dev  cbull-dev  Synced  Healthy        ingress.extensions/cbull-dev unchanged
{{< /highlight >}}

## Finishing up

Now that we've switched to defining the state of our manifests in git, we need to find a way to update the image reference when we release updates.

While there may be a more elegant way to achieve this, I made a simple deployment template with a filler value in the image reference and use sed to generate the final deployment spec.

{{< highlight yaml >}}
    containers:
      - name: cbull-dev
        image: csbull55/cbull-dev:SED_imageName
{{< /highlight >}}

Let's add a quick deployment job on the app repository to generate the manifest and commit it to our infra repo.

{{< highlight yaml >}}
name: deploy

on:
  workflow_run:
    workflows: ["buildAndPush"]
    branches: [main]
    types:
      - completed

jobs:
  on-success:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
    
      - name: Checkout
        uses: actions/checkout@v2
        with:
          repository: Christian-Bull/cbull-dev-infra
          token: ${{ secrets.CBULL_PAT }}
        
      - name: set env vars
        run: |
          echo "SHA=${GITHUB_SHA}" >> $GITHUB_ENV
          echo "GITHUB_REF_NAME=${GITHUB_REF_NAME}" >> $GITHUB_ENV
          
      - name: update deployment spec
        run: sed 's/SED_imageName/${{ env.GITHUB_REF_NAME }}-${{ env.SHA }}/g' manifests/.templates/deployment.yml > manifests/deployment.yml

      - name: Create Pull Request
        uses: EndBug/add-and-commit@v7
        with:
          branch: main
{{< /highlight >}}
