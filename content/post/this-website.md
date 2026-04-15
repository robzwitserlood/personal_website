---
title: How I deployed this website
draft: False
date: 2026-01-01
---

So as a first post, lets briefly explain how I deployed this website. I used [Hugo](https://gohugo.io/) to build the website, and [Scaleway Object Storage](https://www.scaleway.com/en/object-storage/) to host it. I chose Hugo because it's a static site generator that allows me to easily create and manage my content, and Scaleway because it's a European cloud provider that offers affordable and reliable object storage.

Most credits for this website go to [this tutorial](https://www.scaleway.com/en/docs/tutorials/deploy-static-website-with-hugo-and-github-runners-to-object-storage/) by Scaleway, which I followed step by step. I also used the [Blowfish theme](https://blowfish.page/) for Hugo, which provides a clean and simple design for my website.

The most interesting part for me is the CI/CD pipeline that I set up using GitHub Actions. Whenever I push changes to the main branch, GitHub Actions automatically builds the website using Hugo and deploys it to Object Storage. This way, I can focus on creating content without worrying about the deployment process. The code snippet for the GitHub Actions workflow looks like this:

```yaml
name: Build and Deploy Hugo

on:
  push:
    branches: [ "main" ]

jobs:
  build_and_deploy:
    name: Build and Deploy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Hugo
        run: |
          set -e
          HUGO_DOWNLOAD=hugo_extended_withdeploy_0.152.2_Linux-64bit.tar.gz
          wget https://github.com/gohugoio/hugo/releases/download/v0.152.2/${HUGO_DOWNLOAD}
          tar xvzf ${HUGO_DOWNLOAD} hugo
          mv hugo $HOME/hugo
          echo "$HOME" >> $GITHUB_PATH
        shell: bash

      - name: Validate AWS secrets
        run: |
          if [ -z "${{ secrets.SCW_ACCESS_KEY_ID }}" ] || [ -z "${{ secrets.SCW_SECRET_ACCESS_KEY }}" ]; then
            echo "ERROR: Missing AWS credentials. Add SCW_ACCESS_KEY_ID and SCW_SECRET_ACCESS_KEY to repository secrets."
            exit 1
          fi
        shell: bash
      
      - name: build site
        run: hugo
        shell: bash
      
      - name: Deploy to Scaleway
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.SCW_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.SCW_SECRET_ACCESS_KEY }}
          AWS_REGION: nl-ams
        run: hugo deploy --force --maxDeletes -1
        shell: bash
```