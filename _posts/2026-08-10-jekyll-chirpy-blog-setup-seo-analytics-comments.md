---
title: "Setting Up a Developer Blog with Jekyll Chirpy — SEO, Analytics, Comments, and Pageviews"
date: 2026-08-10
categories: [DevOps, Blogging]
tags: [jekyll, chirpy, github-pages, seo, google-analytics, goatcounter, giscus, blogging, developer-tools, static-site]
description: "Complete guide to setting up a developer blog using Jekyll Chirpy theme on GitHub Pages. Covers SEO optimization, Google Analytics, GoatCounter pageviews, Giscus comments, sitemap configuration, and deployment — with code snippets and config examples."
image:
  path: /assets/img/posts/coding_6mjf.svg
  alt: Setting Up a Developer Blog with Jekyll Chirpy
---

## Why This Guide?

Starting a tech blog shouldn't take days of research. I recently set up this blog using Jekyll with the Chirpy theme on GitHub Pages — free hosting, custom domain support, and zero server management.

This post documents the exact steps: from first deploy to having a fully functional blog with SEO, analytics, comments, and pageview counters. Everything you need as a starting point.

---

## The Stack

| Component | Tool | Cost |
|-----------|------|------|
| Static Site Generator | [Jekyll](https://jekyllrb.com/) | Free |
| Theme | [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) | Free |
| Hosting | [GitHub Pages](https://pages.github.com/) | Free |
| Analytics | [Google Analytics](https://analytics.google.com/) | Free |
| Search Indexing | [Bing Webmaster Tools](https://www.bing.com/webmasters) | Free |
| Pageview Counter | [GoatCounter](https://www.goatcounter.com/) | Free (personal) |
| Comments | [Giscus](https://giscus.app/) | Free |
| CI/CD | [GitHub Actions](https://github.com/features/actions) | Free |

Total cost: **$0/month**

---

## Step 1: Create Your Blog Repository

Use the Chirpy starter template:

1. Go to [chirpy-starter](https://github.com/cotes2020/chirpy-starter) and click **"Use this template"**
2. Name the repo `<yourusername>.github.io`
3. Clone it locally:

```bash
git clone https://github.com/<yourusername>/<yourusername>.github.io.git
cd <yourusername>.github.io
```

Install dependencies and run locally:

```bash
bundle install
bundle exec jekyll serve
```

Your site is now live at `http://localhost:4000`.

---

## Step 2: Configure `_config.yml`

This is the heart of your blog. Here's the essential configuration:

```yaml
theme: jekyll-theme-chirpy

lang: en
timezone: Asia/Singapore  # Change to your timezone

title: Your Name
tagline: Your tagline here
description: >-
  A brief description for SEO purposes. This appears in search results.

url: "https://yourusername.github.io"
baseurl: ""

github:
  username: yourusername

social:
  name: Your Name
  email: your@email.com
  links:
    - https://github.com/yourusername
    - https://linkedin.com/in/yourprofile

avatar: https://avatars.githubusercontent.com/yourusername

toc: true
paginate: 10
```

> Replace all placeholder values with your own details. The `url` field is critical for SEO and sitemap generation.
{: .prompt-warning }

---

## Step 3: Write Your First Post

Create a file in `_posts/` with the naming format:

```
YYYY-MM-DD-title-of-your-post.md
```

> The date in the filename is mandatory for Jekyll posts. It won't appear in your URL if you use the `:title` permalink.
{: .prompt-info }

Front matter template:

```yaml
---
title: "Your Post Title"
date: 2026-08-10
categories: [Category1, Category2]
tags: [tag1, tag2, tag3]
description: "One-line description for SEO and social sharing."
image:
  path: /assets/img/posts/your-image.svg
  alt: Descriptive alt text
---

Your content here in Markdown...
```

**Tips for front matter:**
- `categories` — use max 2 levels (parent, child)
- `tags` — use lowercase, hyphenated, relevant keywords
- `description` — keep it under 160 characters for search results
- `image` — appears as the post thumbnail and social preview

---

## Step 4: Deploy with GitHub Actions

Create `.github/workflows/pages-deploy.yml`:

```yaml
name: "Build and Deploy"
on:
  push:
    branches:
      - main
    paths-ignore:
      - .gitignore
      - README.md
      - LICENSE

  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.3"
          bundler-cache: true

      - name: Build site
        run: bundle exec jekyll build
        env:
          JEKYLL_ENV: production

      - name: Upload site artifact
        uses: actions/upload-pages-artifact@v4
        with:
          path: "_site"

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Then in your repo settings:
- Go to **Settings → Pages**
- Set source to **GitHub Actions**

Every push to `main` will now auto-build and deploy.

---

## Step 5: SEO Setup

### Google Search Console Verification

1. Go to [Google Search Console](https://search.google.com/search-console)
2. Add your property: `https://yourusername.github.io`
3. Choose **HTML tag** verification and copy the content value
4. Add it to `_config.yml`:

```yaml
webmaster_verifications:
  google: "your-verification-code-here"
```

### Sitemap

The `jekyll-sitemap` gem auto-generates a sitemap at `/sitemap.xml`. Add it to your `Gemfile`:

```ruby
group :jekyll_plugins do
  gem "jekyll-sitemap"
end
```

Submit your sitemap URL in Google Search Console:

```
https://yourusername.github.io/sitemap.xml
```

### Sitemap Optimization — Exclude Thin Pages

New blogs with few posts generate many thin tag/category pages. Exclude them from the sitemap until you have enough content:

```yaml
defaults:
  - scope:
      path: ""
      type: tags
    values:
      sitemap: false
  - scope:
      path: ""
      type: categories
    values:
      sitemap: false
```

> A lean sitemap with only quality pages helps Google focus its crawl budget on your actual content.
{: .prompt-tip }

### Exclude Non-Content Files

If you have verification HTML files (like `google*.html`), exclude them from the sitemap by adding front matter:

```yaml
---
sitemap: false
---
```

### Bing Webmaster Tools

Bing also powers Yahoo and DuckDuckGo results, so submitting here covers 3 search engines at once.

1. Go to [Bing Webmaster Tools](https://www.bing.com/webmasters)
2. Sign in with your Microsoft account
3. Add your site: `https://yourusername.github.io`
4. Choose the **XML file** verification method — download `BingSiteAuth.xml`
5. Place it in your repo root
6. Also add the verification code to `_config.yml`:

```yaml
webmaster_verifications:
  google: "your-google-verification-code"
  bing: "your-bing-verification-code"
```

After deploying, click **Verify** in Bing Webmaster Tools, then submit your sitemap:

```
https://yourusername.github.io/sitemap.xml
```

> Bing indexes slower than Google but is worth setting up early — it's zero effort once configured.
{: .prompt-tip }

---

## Step 6: Google Analytics

Set up tracking to understand your audience:

1. Create a property at [analytics.google.com](https://analytics.google.com)
2. Get your Measurement ID (starts with `G-`)
3. Add to `_config.yml`:

```yaml
analytics:
  google:
    id: G-XXXXXXXXXX
```

That's it. Chirpy injects the tracking script automatically. You'll see data in your GA dashboard within 24–48 hours.

---

## Step 7: GoatCounter Pageviews

Display a view count on each post — lightweight, privacy-friendly, no cookie banner needed.

1. Create a free account at [goatcounter.com](https://www.goatcounter.com)
2. Note your site code (the subdomain, e.g., `yoursite`)
3. Add both the analytics tracking and the pageview display config:

```yaml
analytics:
  google:
    id: G-XXXXXXXXXX
  goatcounter:
    id: yoursite

pageviews:
  provider: goatcounter
```

> GoatCounter serves dual purpose here: it tracks visits AND provides the view count that Chirpy displays on each post.
{: .prompt-tip }

The view count appears automatically next to the post date — no custom code needed.

---

## Step 8: Giscus Comments

Add a GitHub-powered comment section to every post using [Giscus](https://giscus.app/).

### Prerequisites

1. Your repo must be **public**
2. Install the [Giscus GitHub App](https://github.com/apps/giscus) on your repo
3. Enable **Discussions** in your repo (Settings → Features → Discussions)

### Get Your Config

Go to [giscus.app](https://giscus.app), enter your repo name, and select your preferences. It generates the required IDs.

### Add to `_config.yml`

```yaml
comments:
  provider: giscus
  giscus:
    repo: yourusername/yourusername.github.io
    repo_id: R_kgDOxxxxxxx
    category: General
    category_id: DIC_kwDOxxxxxxx
    mapping: pathname
    strict: 0
    input_position: top
    lang: en
    reactions_enabled: 1
```

Comments now appear at the bottom of every post. Readers authenticate with their GitHub account to comment.

> Giscus automatically matches your site's dark/light theme. No extra configuration needed.
{: .prompt-info }

---

## Complete `_config.yml` Reference

Here's everything together — copy this as your starting template:

```yaml
theme: jekyll-theme-chirpy

lang: en
timezone: Asia/Singapore

# Site identity
title: Your Name
tagline: Your tagline
description: >-
  Your SEO description here.

url: "https://yourusername.github.io"
baseurl: ""

# Social
github:
  username: yourusername

social:
  name: Your Name
  email: you@email.com
  links:
    - https://github.com/yourusername

# Verification
webmaster_verifications:
  google: "your-google-verification-code"

# Appearance
avatar: https://avatars.githubusercontent.com/yourusername
toc: true

# Analytics
analytics:
  google:
    id: G-XXXXXXXXXX
  goatcounter:
    id: yoursite

# Pageviews on posts
pageviews:
  provider: goatcounter

# Comments
comments:
  provider: giscus
  giscus:
    repo: yourusername/yourusername.github.io
    repo_id: R_kgDOxxxxxxx
    category: General
    category_id: DIC_kwDOxxxxxxx
    mapping: pathname
    strict: 0
    input_position: top
    lang: en
    reactions_enabled: 1

# Pagination
paginate: 10
```

---

## Checklist Before Going Live

| # | Item | Status |
|---|------|--------|
| 1 | `_config.yml` configured with your details | |
| 2 | First post published in `_posts/` | |
| 3 | GitHub Actions deploying successfully | |
| 4 | Google Search Console verified | |
| 5 | Sitemap submitted to Search Console | |
| 6 | Google Analytics tracking confirmed | |
| 7 | GoatCounter pageviews showing | |
| 8 | Giscus comments working | |
| 9 | Thin pages excluded from sitemap | |
| 10 | Avatar and social links set | |

---

## Common Pitfalls

### Posts not showing up?
- Filename must be `YYYY-MM-DD-title.md` — no exceptions
- Date must not be in the future (relative to your timezone setting)
- File must be in `_posts/` directory

### Sitemap bloated with tag pages?
- Add `sitemap: false` defaults for tags/categories as shown above
- Wait until you have 3+ posts per tag before including them

### Comments not loading?
- Verify the Giscus app is installed on your repo
- Discussions must be enabled in repo settings
- Check `repo_id` and `category_id` are correct

### Analytics not tracking?
- Wait 24–48 hours for Google Analytics to show data
- Use GA's Realtime report to verify immediately
- Check browser ad-blockers aren't blocking the script

---

## Useful Links

| Resource | URL |
|----------|-----|
| Chirpy Theme Docs | [chirpy.cotes.page](https://chirpy.cotes.page/) |
| Chirpy Starter Template | [github.com/cotes2020/chirpy-starter](https://github.com/cotes2020/chirpy-starter) |
| Jekyll Documentation | [jekyllrb.com/docs](https://jekyllrb.com/docs/) |
| GitHub Pages Docs | [docs.github.com/pages](https://docs.github.com/en/pages) |
| Google Search Console | [search.google.com/search-console](https://search.google.com/search-console) |
| Google Analytics | [analytics.google.com](https://analytics.google.com) |
| GoatCounter | [goatcounter.com](https://www.goatcounter.com) |
| Giscus | [giscus.app](https://giscus.app) |
| unDraw (free SVGs) | [undraw.co](https://undraw.co/) |
| Jekyll Sitemap Plugin | [github.com/jekyll/jekyll-sitemap](https://github.com/jekyll/jekyll-sitemap) |

---

## What's Next?

Once your blog is live with all these features, focus on:

- **Writing consistently** — even one post a month builds SEO authority
- **Sharing on LinkedIn/Twitter** — drives initial traffic before organic search kicks in
- **Requesting indexing** in Search Console for each new post
- **Monitoring GoatCounter** — see which topics resonate with readers

The technical setup is the easy part. The hard part is writing. Start now.

---

## Related Posts

- [Java 18 to Java 21 Migration Guide — Features, Code Examples, Checklist](/posts/java-18-to-java-21-migration/) — Upgrade to the latest LTS with virtual threads, pattern matching, and sequenced collections.
- [Java Collections + Lambda Expressions — Practical Guide with Code Examples](/posts/java-collections-lambda-simplified-development/) — Master streams, filtering, sorting, and grouping with real before/after examples.
