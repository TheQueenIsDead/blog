# Sites

## Overview

### Paper

Paper is a `blog` which will (potentially) have many random writeup of my various shenanigans.

It uses a theme called [Papermod](https://themes.gohugo.io/themes/hugo-papermod/) 

#### Posts

Create a new page by using hugo

```shell
hugo new posts/<name>.md
```

A full range of configuration options for the theme can be found [here](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation#sample-pagemd)

### LZ

LZ is a `landing zone` for displaying social links, it uses a neat theme called [Lynx](https://themes.gohugo.io/themes/lynx/).

## Developing 

[//]: # (TODO)

## Deployment

Github pages used to heavily rely on serving your site based on a specific branch name, but has
[since moved to Github Actions](https://github.blog/changelog/2022-07-27-github-pages-custom-github-actions-workflows-beta/).

Along with the change has come a recommended set of [starter workflows](https://github.com/actions/starter-workflows/blob/main/pages/hugo.yml)
curated by the wonderful people at Github. The Hugo workflow builds and publishes the static content with a Github action instead.

## TODO?

Schema for resume?

https://jsonresume.org/schema/