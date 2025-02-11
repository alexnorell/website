---
title: "Switching to Netlify for static hosting"
date: 2020-08-16T12:59:51-07:00
draft: false
categories:
  - website
tags:
  - hugo
  - netlify
---

Previously I had my website hosted in a GCP instance behind nginx. That really was too much overhead for what I actually needed for this site.

A couple of the downsides:

1. Manual setup of the entire GCP instance
1. No CI pipeline
1. No CDN
1. Expensive


## Manual Set Up

I go over in my previous post, [Creating and Deploying This Website](../creating-and-deploying-this-website). In summary, I was previously using the free tier of GCP to host a nano sized instance that contained nGINX and several static websites. I needed to set up certbot manually for each of these websites. I also needed to set up users and SSH credentials to lock down the machine.

Other aspects of the server that need to be taken care of on a continual basis:

1. Security Updates
2. Software Updates
3. Users

Just a ton a manual set up.

## No CI pipeline

I didn't have a pipeline set up for my personal website. It would have been easy enough to do with a GitHub action. Build with Hugo and then rsync the content to the appropriate directory in the server. I have actually set it up for a different website that I hosted on the instance.

### GitHub action used to deploy

```yaml
name: Deploy

on:
  push:
    branches:
      - master

jobs:
  Deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Checkout submodules
      shell: bash
      run: |
        git config --global url."https://github.com/".insteadOf "git@github.com:"
        auth_header="$(git config --local --get http.https://github.com/.extraheader)"
        git submodule sync --recursive
        git -c "http.extraheader=$auth_header" -c protocol.version=2 submodule update --init --force --recursive --depth=1
    - name: Setup Hugo
      uses: peaceiris/actions-hugo@v2
      with:
        hugo-version: "latest"
    - name: Build site
      run: hugo --minify
    - name: Deploy to server
      uses: Pendect/action-rsyncer@v1.1.0
      env:
        DEPLOY_KEY: ${{secrets.DEPLOY_KEY}}
      with:
        flags: "-avzO --no-o --no-g --delete"
        options: ""
        ssh_options: ""
        src: "public/"
        dest: "github@example.com:/var/www/example.com/"
    - name: Display status from deploy
      run: echo "${{ steps.deploy.outputs.status }}"

```

It was easy to do, but it wasn't totally fool-proof. I needed to create a user an an SSH key and add the private key to GitHub as a secret.

## No CDN

The websites that were hosted in the instance were only available if the instance was up, and only from it. If someone was trying to access it from Japan or Ukraine, they had to get the content from the instance I was running in the Google Data Center in Oregon. A content deliver network, or CDN, propagates the content of static assets across the internet to hopefully be closer to the requester, making page load times slower. It also has the added benefit of not having a single source of failure. If for some reason one of the data centers is down that is hosting the content of my website, there are other pages in the world that also have the exact same content. The CDN will automatically route the requester to the place where the data is currently available.

## Expensive

For what I'm doing with GCP, I have no reason to actually pay for it. I've had virtual private servers in the past, cheap virtual machines hosted in the cloud, and also used them for web hosting. That was also terribly inefficient. Something like GitHub Pages or Netlify are completely free, and a much better service than what I could do myself. While I did learn a lot when hosting and managing the servers in the past, I don't see any benefit from doing so now. At >$5 a month for an instance in GCP or less if I go with something from [lowendbox.com](https://lowendbox.com), I would much rather host a dynamic service that I can't get for free from another provider.

## Setting it up

I followed the guide on the [Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/). It really took less than 5 minutes to take the content from my [website source repository](https://github.com/alexnorell/website) and get it up in the cloud, hosted by Netlify.
