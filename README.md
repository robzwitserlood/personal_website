# Personal Website

Source code for [robzwitserlood's](https://github.com/robzwitserlood) personal website — a Hugo static site built with the [Blowfish](https://blowfish.page) theme.

## Stack

- **Framework:** [Hugo](https://gohugo.io) (extended, v0.152.2)
- **Theme:** [Blowfish](https://blowfish.page)
- **Hosting:** Scaleway Object Storage (S3-compatible)
- **CI/CD:** GitHub Actions — auto-deploys on push to `main`

## Project Structure

```
content/
  _index.md          # Homepage / about section
  about.md           # About page
  post/              # Blog posts / project write-ups
config/_default/     # Hugo configuration (hugo.toml, params.toml, etc.)
layouts/             # Custom layout overrides
static/              # Static assets served as-is
assets/              # Assets processed by Hugo pipelines
.github/workflows/   # CI/CD pipeline
```

## Local Development

**Prerequisites:** Hugo extended (v0.152.2+)

```bash
# Install Hugo (example via tarball — match the version in the workflow)
wget https://github.com/gohugoio/hugo/releases/download/v0.152.2/hugo_extended_withdeploy_0.152.2_Linux-64bit.tar.gz
tar xvzf hugo_extended_withdeploy_*.tar.gz hugo
sudo mv hugo /usr/local/bin/

# Serve locally with live reload
hugo server

# Build for production
hugo
```

The local dev server runs at `http://localhost:1313` by default.

## Deployment

Deployment happens automatically via GitHub Actions on every push to `main`:

1. Hugo builds the site into `public/`
2. `hugo deploy` syncs the output to a Scaleway S3 bucket

To deploy manually, configure the following environment variables and run:

```bash
export AWS_ACCESS_KEY_ID=<SCW_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<SCW_SECRET_ACCESS_KEY>
export AWS_REGION=nl-ams
hugo deploy --force --maxDeletes -1
```

The required secrets (`SCW_ACCESS_KEY_ID`, `SCW_SECRET_ACCESS_KEY`) must be set in the repository's GitHub Actions secrets for automated deploys to work.

## Adding Content

New posts go under `content/post/<slug>/index.md`. Use an existing post as a reference for front matter fields (title, date, tags, description, etc.).
