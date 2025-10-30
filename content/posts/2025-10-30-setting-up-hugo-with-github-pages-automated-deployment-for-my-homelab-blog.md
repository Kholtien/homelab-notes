---
title: "Setting Up Hugo with GitHub Pages: Automated Deployment for My Homelab Blog"
date: 2025-10-30T13:35:22+11:00
draft: true
tags: ["hugo", "github-pages", "blogging", "cicd", "github-actions"]
---

## Introduction

I've been wanting to document my homelab projects for a while now. Like many sysadmins, I have a collection of scattered notes, random markdown files, and plenty of "I should write that down" moments that never quite make it to any permanent storage. I decided it was time to set up a proper blog to document these adventures.

After exploring options, I settled on Hugo for the static site generator and GitHub Pages for hosting. The combination offers exactly what I need: complete control over my content, version control built in, free hosting, and fast load times. Plus, being a static site means I don't have to worry about securing a CMS or dealing with database backups.

This post documents the entire setup process—from choosing the approach to getting the site live with automated deployments. I'm documenting this as I go, which means you'll see both what worked smoothly and the hiccups I encountered along the way.

## Background and Decision-Making

### Why Hugo?

I looked at several static site generators: Jekyll (GitHub's default), Gatsby, Next.js, and Hugo. I chose Hugo primarily for its speed and simplicity. It's a single binary with no dependency management (no Node modules, no Ruby gems), builds are incredibly fast, and it has a strong ecosystem of themes designed for technical blogs.

The fact that it's written in Go and produces a standalone binary appealed to my sysadmin sensibilities—I like tools that just work without complex setup.

### Why GitHub Pages?

For hosting, GitHub Pages was the obvious choice:
- **Free hosting** for public repositories
- **Custom domain support** with free SSL via Let's Encrypt
- **Version control integration** - my content is already in Git
- **CDN included** - GitHub's infrastructure handles distribution
- **No server maintenance** - I maintain enough servers in my homelab already

I could have self-hosted this on my own infrastructure, but that would miss the point. I want to document my homelab projects, not create more maintenance work. Using GitHub Pages means one less thing to monitor and maintain.

### The Deployment Strategy

GitHub Pages offers two main deployment approaches:

1. **Branch-based**: Build locally, commit to a `gh-pages` branch
2. **GitHub Actions**: Automated builds on every push to main

I went with GitHub Actions. The branch-based approach means committing built artifacts to version control (never a good idea), and it requires remembering to build before every push. GitHub Actions handles everything automatically—I push my markdown, and within a minute or two, the site updates. Perfect.

## Prerequisites

Before starting this setup, here's what I had ready:

- **A custom domain**: homelab-notes.milliard.au (I already own milliard.au)
- **Git and Hugo installed locally**: I'm on Arch Linux, so `sudo pacman -S hugo git`
- **A GitHub account**: For hosting the repository
- **Basic understanding of**: Git workflows, YAML configuration, and Hugo's structure
- **Time**: About an hour for the full setup (less if you don't count time spent reading documentation)

My environment:
- **Local development**: Arch Linux with Hugo v0.152.2 extended
- **Repository**: github.com/Kholtien/homelab-notes
- **Theme**: PaperMod (clean, fast, great for technical content)

## The Implementation Journey

### Starting with Hugo Configuration

I already had a basic Hugo site initialized with the PaperMod theme as a git submodule. The initial `hugo.toml` configuration was pretty bare-bones—just the example URL and basic title. This needed updating for the actual deployment.

The first thing I did was update the base configuration to reflect my actual site:

```toml
# Site configuration for Homelab Notes
# This Hugo site is deployed to GitHub Pages with a custom domain

baseURL = 'https://homelab-notes.milliard.au/'
languageCode = 'en-au'
title = 'Homelab Notes'
theme = 'PaperMod'
```

The `baseURL` is critical—Hugo uses this for generating absolute URLs in the built site. Getting this wrong means broken links and missing assets. I also set `languageCode` to `en-au` because Australian English is the correct English (fight me).

### Configuring the PaperMod Theme

Next, I added the PaperMod theme configuration. PaperMod has excellent defaults, but I wanted to customize a few things:

```toml
# Enable emoji support in content
enableEmoji = true

# Pagination settings
[pagination]
  pagerSize = 10

# PaperMod theme parameters
[params]
  description = "Self-hosting documentation and homelab experiments"
  author = "Colton"

  # Show reading time estimate on posts
  ShowReadingTime = true

  # Show table of contents on posts
  ShowToc = true
  TocOpen = false

  # Add copy buttons to code blocks
  ShowCodeCopyButtons = true

  # Date format for posts (Australian format: day month year)
  DateFormat = "2 January 2006"
```

A quick note on that date format: Hugo uses Go's time formatting, which has a quirky but memorable system. Instead of format specifiers like `%d`, you use a reference date (January 2, 2006 at 3:04:05 PM MST, which is 01/02 03:04:05PM '06 -0700). So "2 January 2006" gives you "30 October 2025" format. I went with day-first because that's the sensible way to write dates.

I also set up the navigation menu and home page info:

```toml
# Home page info section
[params.homeInfoParams]
  Title = "Homelab Notes"
  Content = "Documentation of self-hosting projects, homelab experiments, and infrastructure adventures."

# Navigation menu
[[menu.main]]
  name = "Posts"
  url = "/posts/"
  weight = 10

[[menu.main]]
  name = "Tags"
  url = "/tags/"
  weight = 20

[[menu.main]]
  name = "About"
  url = "/about/"
  weight = 30
```

### Build Performance Configuration

One thing I learned from the Hugo documentation: proper caching configuration is important for GitHub Actions. Hugo can cache various resources between builds, significantly speeding things up:

```toml
# Caching configuration for faster builds
# Required for GitHub Actions deployment
[caches]
  [caches.images]
    dir = ':cacheDir/images'
  [caches.assets]
    dir = ':cacheDir/assets'
  [caches.getresource]
    dir = ':cacheDir/getresource'

# Minification settings for production builds
[minify]
  [minify.tdewolff]
    [minify.tdewolff.html]
      keepWhitespace = false
```

The `:cacheDir` syntax tells Hugo to use its standard cache directory, which GitHub Actions can persist between runs.

### The Pagination Gotcha

While testing the configuration, I hit my first error:

```
ERROR deprecated: site config key paginate was deprecated in Hugo v0.128.0
and subsequently removed. Use pagination.pagerSize instead.
```

I had initially used the old `paginate = 10` syntax from an older tutorial. Hugo has been actively modernizing its configuration format, so some older examples online don't work with recent versions. The fix was simple—replace the old syntax with the new nested structure:

```toml
[pagination]
  pagerSize = 10
```

This is a good reminder to always check the version of documentation you're reading. Hugo's official docs are excellent and always current.

### Creating the GitHub Actions Workflow

Now for the important part: automating the deployment. I created `.github/workflows/hugo.yaml` to handle building and deploying the site.

GitHub Actions workflows use YAML configuration to define what happens when certain events occur (like pushing to a branch). Here's the complete workflow I set up:

```yaml
# GitHub Actions workflow for deploying Hugo site to GitHub Pages
# This workflow automatically builds and deploys the site whenever changes are pushed to main

name: Deploy Hugo site to Pages

on:
  # Trigger on pushes to the main branch
  push:
    branches:
      - main

  # Allow manual workflow dispatch from Actions tab
  workflow_dispatch:

# Set permissions for the GitHub token
# These are required for deploying to GitHub Pages
permissions:
  contents: read      # Read repository contents
  pages: write        # Write to GitHub Pages
  id-token: write     # Write ID tokens for deployment

# Ensure only one deployment runs at a time
# If a new deployment starts, cancel any in-progress deployments
concurrency:
  group: "pages"
  cancel-in-progress: false

# Use bash as the default shell for all steps
defaults:
  run:
    shell: bash

jobs:
  # Build job - compiles the Hugo site
  build:
    runs-on: ubuntu-latest
    env:
      # Hugo version to use - matches official Hugo docs recommendation
      HUGO_VERSION: 0.139.3
    steps:
      # Step 1: Install Hugo CLI (extended version for SCSS/Sass support)
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb

      # Step 2: Install Dart Sass for SCSS compilation
      # Note: Remove this step if your theme doesn't use Sass
      - name: Install Dart Sass
        run: sudo snap install dart-sass

      # Step 3: Check out the repository code
      # fetch-depth: 0 gets full history for .GitInfo and .Lastmod
      # submodules: recursive ensures theme submodule is fetched
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      # Step 4: Configure GitHub Pages settings
      # This sets up the Pages URL and other deployment parameters
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      # Step 5: Install Node.js dependencies if package-lock.json exists
      # This is optional and only runs if you have Node dependencies
      - name: Install Node.js dependencies
        run: |
          [[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true

      # Step 6: Build the Hugo site
      # --gc: Run garbage collection after build
      # --minify: Minify HTML, CSS, JS for smaller file sizes
      # --baseURL: Use the Pages URL from the setup step
      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      # Step 7: Upload the built site as an artifact
      # The public/ directory contains the generated static site
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # Deploy job - publishes the built site to GitHub Pages
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build  # This job waits for the build job to complete
    steps:
      # Deploy the artifact to GitHub Pages
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Let me break down the key parts of this workflow:

**Triggers**: The workflow runs on pushes to main, plus I added `workflow_dispatch` so I can manually trigger it from GitHub's interface if needed.

**Permissions**: GitHub Actions needs explicit permissions to deploy to Pages. The three permissions here allow reading the repository, writing to Pages, and creating identity tokens for deployment.

**Concurrency control**: The concurrency group ensures only one deployment runs at a time. If you push multiple commits quickly, they queue up rather than trying to deploy simultaneously.

**The build job**: This is where the magic happens. It installs Hugo (the extended version, which includes Sass support for the theme), checks out the code including the PaperMod submodule, and builds the site with garbage collection and minification enabled.

**The deploy job**: Once the build succeeds, this job takes the built artifacts and deploys them to GitHub Pages. It's marked with `needs: build`, creating a dependency—deployment only happens if the build succeeds.

### Setting Up the Custom Domain

For GitHub Pages to serve my site at homelab-notes.milliard.au instead of kholtien.github.io/homelab-notes, I needed to create a CNAME file:

```bash
mkdir -p static
echo "homelab-notes.milliard.au" > static/CNAME
```

Hugo copies everything in the `static/` directory to the root of the built site, so this CNAME file ends up in `public/CNAME` after the build, which is exactly what GitHub Pages needs.

## Testing Before Deployment

Before pushing everything to GitHub, I wanted to make sure the site actually built correctly. Hugo makes this easy:

```bash
hugo --gc --minify
```

The first attempt worked perfectly—no errors, clean build output:

```
Start building sites …
hugo v0.152.2+extended+withdeploy linux/amd64 BuildDate=unknown

                  | EN
──────────────────┼────
 Pages            | 10
 Paginator pages  │  0
 Non-page files   │  0
 Static files     │  1
 Processed images │  0
 Aliases          │  2
 Cleaned          │  0

Total in 36 ms
```

Building in 36 milliseconds. This is why I like Hugo.

I also verified the CNAME file made it into the build output:

```bash
cat public/CNAME
# Output: homelab-notes.milliard.au
```

Perfect. Everything looked good locally, so it was time to push to GitHub.

## Deployment and DNS Configuration

### Pushing to GitHub

I committed all the changes:

```bash
git add .github/workflows/hugo.yaml hugo.toml static/CNAME
git commit -m "Configure GitHub Actions deployment and custom domain"
git push origin main
```

The push triggered the GitHub Actions workflow immediately. I watched it run in the Actions tab—the build completed in about 90 seconds.

[Screenshot needed: GitHub Actions workflow running successfully with green checkmarks showing all steps completed]
![GitHub Actions Workflow Success](placeholder-screenshot.svg)

### Configuring GitHub Pages Settings

After the first successful workflow run, I needed to configure GitHub Pages in the repository settings:

1. Navigate to the repository on GitHub
2. Go to Settings → Pages
3. Under "Build and deployment", set Source to "GitHub Actions"
4. The custom domain field shows homelab-notes.milliard.au (from the CNAME file)
5. Wait for DNS check to complete
6. Enable "Enforce HTTPS" once DNS verifies

[Screenshot needed: GitHub Pages settings page showing GitHub Actions as source and custom domain configured]
![GitHub Pages Configuration](placeholder-screenshot.svg)

### DNS Configuration

For the custom domain to work, I needed to add a CNAME record in my DNS provider (Cloudflare):

```
Type: CNAME
Name: homelab-notes
Value: kholtien.github.io
TTL: Auto
Proxy status: DNS only (orange cloud off)
```

The CNAME record points homelab-notes.milliard.au to kholtien.github.io, and GitHub Pages handles the rest. I turned off Cloudflare's proxy (the orange cloud) because GitHub Pages needs to see the original hostname to serve the correct site.

DNS propagation was nearly instant for me, but it can take up to 24 hours in some cases.

[Screenshot needed: DNS record configuration in Cloudflare showing the CNAME record]
![DNS Configuration](placeholder-screenshot.svg)

## Validation and First Real Post

Once DNS propagated, I visited https://homelab-notes.milliard.au and saw... my site! The SSL certificate was automatically provisioned by GitHub (using Let's Encrypt), navigation worked, and the PaperMod theme looked clean.

[Screenshot needed: The live site homepage showing the Homelab Notes title and clean PaperMod theme]
![Live Site Homepage](placeholder-screenshot.svg)

The whole deployment workflow works like this now:

1. I write a post in `content/posts/` as a markdown file
2. Set `draft: false` when ready to publish
3. Commit and push to GitHub
4. GitHub Actions builds the site (takes about 60-90 seconds)
5. The site updates automatically

No manual building, no separate deployment steps, no SSH'ing into servers. Just push and wait a minute.

## Lessons Learned

### What Worked Really Well

**Hugo's speed**: Building the entire site in 36ms locally means the development feedback loop is instant. Running `hugo server` gives you live-reloading—I can see changes immediately as I write.

**GitHub Actions**: The automation is seamless. I was worried about complex CI/CD configuration, but the official Hugo workflow template worked perfectly. No debugging required.

**PaperMod theme**: The theme's defaults are excellent for technical content. Code blocks look good, the table of contents is helpful, and copy buttons on code blocks are a nice touch.

**Configuration as code**: Everything about this site is version-controlled. If something breaks, I can roll back. If I want to experiment, I can create a branch. This is how infrastructure should work.

### Things That Surprised Me

**The pagination deprecation**: I expected the Hugo configuration format to be stable, but it's actively evolving. Not a bad thing—the new format is cleaner—but worth noting that older tutorials might need adjustment.

**DNS propagation speed**: I expected to wait hours for DNS to propagate. In practice, my site was accessible within 5 minutes. Cloudflare's DNS is fast.

**GitHub Pages custom domain SSL**: I thought I'd need to manage certificates manually. GitHub provisions and renews Let's Encrypt certificates automatically. One less thing to worry about.

### What I'd Do Differently

If I were starting fresh, I'd create a more detailed `.gitignore` earlier. I initially committed the Hugo build lock file (`.hugo_build.lock`) before realizing it should be ignored. Not a major issue, but cleaner to handle it upfront.

I'd also test the Hugo build locally *before* writing the GitHub Actions workflow. I got lucky that my configuration worked first try, but catching issues locally is faster than debugging in CI/CD.

### Future Improvements

A few things I'm planning to add:

**About page**: I have it in the menu, but haven't written the content yet. Need to document what this blog is about and who I am.

**Analytics (maybe)**: I'm torn on this. Part of me wants to know if anyone reads this. Another part likes the privacy of not tracking visitors. If I add it, it'll be privacy-respecting (probably Plausible or Umami).

**Search functionality**: PaperMod supports search, but it requires additional setup. As I write more posts, search will become more valuable.

**Automated screenshot optimization**: Right now I'll be adding screenshots manually. Setting up automatic image optimization in the GitHub Actions workflow could save bandwidth.

## Troubleshooting Guide

Here are issues I encountered (or anticipated) and their solutions:

### Build Fails: Hugo Version Mismatch

**Problem**: GitHub Actions uses Hugo 0.139.3, but your local version is different, causing inconsistent builds.

**Solution**: Either update your local Hugo to match, or change `HUGO_VERSION` in the workflow file to match your local version. Keep them in sync.

### Custom Domain Shows 404

**Problem**: Site builds successfully but shows 404 at custom domain.

**Solution**:
1. Verify CNAME file exists in `static/CNAME`
2. Check DNS record points to `kholtien.github.io` (your-username.github.io)
3. Verify `baseURL` in hugo.toml matches your custom domain exactly
4. Wait up to 24 hours for DNS propagation

### Theme Not Loading

**Problem**: Site loads but has no styling.

**Solution**:
1. Ensure theme is checked out: `git submodule update --init --recursive`
2. Verify `theme = 'PaperMod'` in hugo.toml
3. Check the Actions workflow has `submodules: recursive` in the checkout step

### SSL Certificate Errors

**Problem**: Browser shows certificate warnings.

**Solution**:
1. Wait—GitHub can take up to 24 hours to provision certificates
2. Ensure DNS is correctly configured (CNAME pointing to github.io domain)
3. Check "Enforce HTTPS" is enabled in GitHub Pages settings

## Conclusion

Setting up Hugo with GitHub Pages turned out to be straightforward once I understood the pieces. The combination gives me a fast, maintainable platform for documenting my homelab projects without adding more infrastructure to manage.

The automated deployment via GitHub Actions is excellent—I can focus on writing content rather than managing build processes. The whole workflow from writing markdown to having content live takes less than two minutes, which is about as good as it gets.

Now I have no excuse not to document my projects. The infrastructure is there, the process is smooth, and I'm writing this post as proof that it works.

Next up: I'll be documenting my Authentik SSO setup, followed by the Pangolin reverse proxy configuration. Stay tuned.

---

## Additional Resources

- [Hugo Documentation](https://gohugo.io/documentation/)
- [Hugo's GitHub Pages Deployment Guide](https://gohugo.io/host-and-deploy/host-on-github-pages/)
- [PaperMod Theme Documentation](https://github.com/adityatelange/hugo-PaperMod/wiki)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

