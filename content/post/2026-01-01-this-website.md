---
title: How I deployed this website
draft: False
date: 2026-01-01
---

So as a first post, lets briefly explain how I deployed this website. I used [Hugo](https://gohugo.io/) to build the website, and [Scaleway Object Storage](https://www.scaleway.com/en/object-storage/) to host it. I chose Hugo because it's a static site generator that allows me to easily create and manage my content, and Scaleway because it's a European cloud provider that offers affordable and reliable object storage.

Most credits for this website go to [this tutorial](https://www.scaleway.com/en/docs/tutorials/deploy-static-website-with-hugo-and-github-runners-to-object-storage/) by Scaleway, which I followed step by step. I also used the [Blowfish theme](https://blowfish.page/) for Hugo, which provides a clean and simple design for my website.

The most interesting part for me is the CI/CD pipeline that I set up using GitHub Actions. Whenever I push changes to the main branch, GitHub Actions automatically builds the website using Hugo and deploys it to Object Storage. This way, I can focus on creating content without worrying about the deployment process. The code snippet for the GitHub Actions workflow looks like this:

{{< codeimporter url="https://raw.githubusercontent.com/robzwitserlood/personal_website/refs/heads/main/.github/workflows/main.yml" type="yml" >}}