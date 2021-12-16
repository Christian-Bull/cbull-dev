# [cbull.dev](https://cbull.dev/)

A simple personal site created using [Hugo](https://gohugo.io/) and [Etch](https://github.com/LukasJoswiak/etch). A super quick static site generator written in [Go](https://go.dev/).

## Overview

`/site/content`  
Contains all custom content

`/site/themes/etch`  
Base etch build with a few extra bits of customization to tweak the base posts page. I'm sure this could have been achieved without editing the theme directly but it's a static site and I'll never upgrade the theme.

`/site/public`  
Output directory after build

`/manifests`  
Contains a bare-bones setup for running this site on a k8s cluster.