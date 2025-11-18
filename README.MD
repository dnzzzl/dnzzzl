# Denzel's Blog

Hugo-based blog deployed to GitHub Pages at https://blog.abcdenzel.xyz

## Quick Start

### Local Development

```bash
# Start Hugo development server
hugo serve

# Start with drafts visible
hugo serve -D

# Build for production
hugo
```

### Deployment

The site automatically deploys to GitHub Pages on push to the `master` branch.

**First-time setup:**

1. In GitHub repository settings:
   - Go to **Settings > Pages**
   - Under "Build and deployment", set **Source** to **GitHub Actions**
2. Configure custom domain:
   - In **Settings > Pages > Custom domain**, enter `blog.abcdenzel.xyz`
3. In your DNS provider (e.g., Cloudflare), add DNS records:
   ```
   Type: CNAME
   Name: blog
   Target: yourusername.github.io
   ```
4. Wait for DNS propagation (can take up to 24 hours, usually ~10 minutes)
5. In **Settings > Pages**, check "Enforce HTTPS" once DNS is working

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions deployment
├── archetypes/                 # Content templates
├── content/
    |
    en/es
│   └── posts/
│       ├── meta/               # Meta posts (blogging, philosophy, etc.)
│   │   └── tech/               # Technical posts
│   ├── about.md                # About page
│   └── author.md               # Author bio
├── themes/
│   └── nightfall/              # Alternative theme (git submodule)
├── hugo.toml                   # Hugo configuration
└── README.md                   # This file
```

## Configuration

Main configuration in `hugo.toml`:

- **baseURL**: `https://blog.abcdenzel.xyz/`
- **Theme**: `diary` (hugo-theme-diary)
- **Hugo version**: 0.152.2+

## Writing Posts

Create a new post:

```bash
hugo new content/posts/tech/my-post-title.md
```

Post frontmatter example:

```toml
+++
date = '2025-11-16T10:30:00-04:00'
draft = false
title = 'My Post Title'
tags = ['tag1', 'tag2']
categories = ['tech']
+++
```

## Themes

Two themes available:

1. **diary** (current): Mobile-friendly journal theme with dark mode


## Resources

- [Hugo Documentation](https://gohugo.io/documentation/)
- [GitHub Pages Docs](https://docs.github.com/en/pages)
- [hugo-theme-diary](https://github.com/AmazingRise/hugo-diary)
- [Nightfall Theme](https://github.com/LordMathis/hugo-theme-nightfall)
