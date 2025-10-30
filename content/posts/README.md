# Posts Directory Structure

This directory contains blog posts using Hugo's **Page Bundle** organization.

## Structure

Each post should be organized as a **page bundle** - a folder containing the post and all its assets:

```
content/posts/
├── 2025-10-30-setting-up-hugo-with-github-pages/
│   ├── index.md              # The blog post content
│   └── images/               # Images specific to this post
│       ├── screenshot1.png
│       ├── screenshot2.png
│       └── diagram.svg
├── 2025-11-01-another-post/
│   ├── index.md
│   └── images/
│       └── example.png
└── README.md                 # This file
```

## Benefits of Page Bundles

1. **Organization**: All post assets live alongside the post content
2. **Portability**: Easy to move or archive entire posts
3. **Relative paths**: Reference images with simple relative paths like `images/screenshot.png`
4. **No conflicts**: Each post's images are isolated from other posts

## Creating a New Post with Images

### Option 1: Manual Creation

```bash
# Create the post directory
POST_DIR="content/posts/$(date +%Y-%m-%d)-my-post-title"
mkdir -p "$POST_DIR/images"

# Create the post file
cat > "$POST_DIR/index.md" << 'EOF'
---
title: "My Post Title"
date: $(date +%Y-%m-%dT%H:%M:%S%z)
draft: true
tags: [tag1, tag2]
---

Content goes here...

![Screenshot description](images/screenshot.png)
EOF
```

### Option 2: Using the Blog Skill Script

```bash
# The blog skill creates page bundles automatically
python3 .claude/skills/hugo-homelab-blog/scripts/new_post.py "My Post Title" --tags tag1 tag2
```

## Adding Images to a Post

1. **Save your screenshots** in the post's `images/` subdirectory:
   ```bash
   # For the Hugo setup post:
   cp ~/Downloads/screenshot.png content/posts/2025-10-30-setting-up-hugo-with-github-pages/images/
   ```

2. **Reference in markdown** using relative paths:
   ```markdown
   ![Descriptive alt text](images/screenshot.png)
   ```

## Image Best Practices

- **Descriptive filenames**: `github-actions-success.png` not `screenshot1.png`
- **Optimize file sizes**: Use compressed PNGs or JPEGs
- **Alt text**: Always provide meaningful alt text for accessibility
- **Reasonable dimensions**: 1200-1600px wide is usually sufficient
- **Supported formats**: PNG, JPEG, SVG, WebP

## Standalone Posts (Legacy)

If you have standalone `.md` files (not in folders), they still work but can't have co-located images. Convert them to page bundles:

```bash
# Convert standalone post to page bundle
POST="content/posts/my-post.md"
POST_DIR="${POST%.md}"
mkdir -p "$POST_DIR/images"
mv "$POST" "$POST_DIR/index.md"
```

## Example: Full Post with Images

```
content/posts/2025-10-30-setting-up-hugo-with-github-pages/
├── index.md
└── images/
    ├── github-actions-success.png      # 143 KB
    ├── github-pages-settings.png       # 87 KB
    ├── cloudflare-dns-config.png       # 62 KB
    └── homelab-notes-homepage.png      # 234 KB
```

**In index.md:**
```markdown
Here's the GitHub Actions workflow:

![GitHub Actions Success](images/github-actions-success.png)

The workflow completed without errors...
```

**Rendered URL:** `/posts/2025-10-30-setting-up-hugo-with-github-pages/images/github-actions-success.png`

---

This organization keeps your content maintainable as your blog grows!
