# anupamsinha.github.io

Personal tech blog — practical Java guides, architecture patterns, and developer references.

**Live at**: [https://anupamsinha.github.io](https://anupamsinha.github.io)

---

## Structure

```
anupamsinha.github.io/
├── _config.yml                  # Site configuration (title, theme, social links, defaults)
├── _posts/                      # Blog posts (markdown files)
│   ├── 2024-08-09-java-18-to-java-21-migration.md
│   └── 2024-08-10-java-collections-lambda-simplified-development.md
├── _tabs/                       # Sidebar navigation pages
│   ├── about.md                 # About page
│   ├── archives.md              # Posts by date
│   ├── categories.md            # Posts by category
│   └── tags.md                  # Posts by tag
├── assets/
│   └── img/
│       └── posts/               # Vector illustrations used in blog posts (SVG)
├── .github/
│   └── workflows/
│       └── pages-deploy.yml     # GitHub Actions workflow for build and deploy
├── index.html                   # Homepage layout
├── Gemfile                      # Ruby dependencies (Jekyll + Chirpy theme)
└── .gitignore                   # Ignored files (_site, vendor, cache, etc.)
```

---

## Theme

Uses [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) — a minimal, responsive Jekyll theme for technical writing.

Features:
- Dark/light mode toggle
- Sidebar navigation
- Table of contents on posts
- Syntax-highlighted code blocks with line numbers
- Categories, tags, and archives
- SEO optimized

---

## Writing a New Post

Create a markdown file in `_posts/` with this naming format:

```
YYYY-MM-DD-title-of-post.md
```

Front matter template:

```yaml
---
title: "Your Post Title"
date: 2024-08-10
categories: [Java, Fundamentals]
tags: [java, spring-boot, microservices]
description: "One-line description for SEO."
image:
  path: /assets/img/posts/your-image.svg
  alt: Image description
---
```

---

## Build and Deploy

The site builds and deploys automatically via GitHub Actions on every push to `main`.

- Workflow: `.github/workflows/pages-deploy.yml`
- Build: Jekyll with `jekyll-theme-chirpy` gem
- Deploy: GitHub Pages (source set to GitHub Actions)

---

## Local Development

```bash
bundle install
bundle exec jekyll serve
```

Site available at `http://localhost:4000`.

---

## Images

Vector illustrations from [unDraw](https://undraw.co/) (free, no attribution required) stored in `assets/img/posts/`.

---

## Topics Covered

- Java (versions, migration, collections, lambda, streams)
- Spring Boot
- Architecture patterns
- Cloud & DevOps
- AI in software engineering
