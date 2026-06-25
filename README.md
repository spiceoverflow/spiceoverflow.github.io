# spiceoverflow.github.io

Personal security research blog by Mo Sakr.

## Deploy to GitHub Pages

1. Create a new repo named `spiceoverflow.github.io` on GitHub.
2. Copy the contents of this `jekyll-site/` folder into the root of that repo.
3. Push to the `main` branch — GitHub Pages builds and deploys automatically.
4. Your site is live at **https://spiceoverflow.github.io** within a minute or two.

## Writing a new post

Create a file in `_posts/` following this naming convention:

```
_posts/YYYY-MM-DD-slug-for-your-post.md
```

Start the file with front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: 2025-07-01
excerpt: "One sentence summary shown on the home page."
---

Your content here in Markdown.
```

## Adding a CVE

Edit `_data/cves.yml` and add an entry:

```yaml
- id: CVE-2025-XXXXX
  title: Short description of the vulnerability
  vendor: affected-project
  severity: High
  date: "2025-07-01"
  url: https://link-to-advisory
```

## Local development (optional)

```bash
gem install bundler
bundle install
bundle exec jekyll serve
```
