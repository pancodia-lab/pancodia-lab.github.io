---
layout: post
title: "How to Set Up a GitHub Pages Blog with Jekyll in 10 Minutes"
date: 2026-03-03
---

You want a dev blog. You don't want to manage a server, pay for hosting, or fight with a CMS. You just want to write markdown, push to GitHub, and have a site.

That's exactly what **GitHub Pages + Jekyll** gives you. GitHub builds and hosts the site for free. You write posts in markdown. No CI pipeline to configure, no Docker containers, no database.

This guide walks through the entire setup — from creating the repo to publishing your first post — based on what I actually did to launch this very site.

## What You'll End Up With

- A blog at `https://<your-org-or-username>.github.io/`
- Posts written in markdown with front matter
- Automatic builds on every `git push`
- A clean, minimal theme (Jekyll's default: Minima)
- Zero ongoing maintenance cost

## Prerequisites

- A GitHub account (personal or organization)
- `git` installed locally
- GitHub CLI (`gh`) installed and authenticated (see [my previous post on gh auth]({% raw %}{{ site.baseurl }}{% endraw %}/2026/03/03/stop-typing-your-github-token/))

## Step 1: Decide on the Repo Name

GitHub Pages has two flavors:

| Type | Repo name | URL |
|---|---|---|
| **User/Org site** | `<username>.github.io` or `<orgname>.github.io` | `https://<name>.github.io/` |
| **Project site** | Any name (e.g., `blog`) | `https://<name>.github.io/blog/` |

**Recommendation:** If this is your main blog, go with the user/org site convention. It gives you the cleanest URL with no path prefix.

For an organization called `my-org`, the repo name would be `my-org.github.io`.

## Step 2: Scaffold the Jekyll Site Locally

You don't need to install Ruby or Jekyll locally — GitHub Pages builds it for you. You just need the right file structure.

### Create the project directory

```bash
mkdir my-org.github.io
cd my-org.github.io
git init
```

### Create `_config.yml`

This is Jekyll's configuration file. At minimum:

```yaml
title: My Dev Blog
description: Technical guides and things I figured out the hard way.
author: Your Name
url: "https://my-org.github.io"

theme: minima

markdown: kramdown
permalink: /:year/:month/:day/:title/
```

Key settings:
- `theme: minima` — GitHub Pages' built-in default theme. Clean and readable. You can [change it later](https://pages.github.com/themes/).
- `permalink` — controls your post URLs. The format above gives you `/2026/03/03/my-post-title/`.
- `url` — must match your actual GitHub Pages URL for links to work correctly.

### Create `index.md`

This is your home page. With the `home` layout, Minima automatically lists your posts here.

```markdown
---
layout: home
title: Home
---
```

That's it. Seriously.

### Create `about.md` (optional)

```markdown
---
layout: page
title: About
permalink: /about/
---

A few words about yourself and what this blog covers.
```

### Create a `Gemfile`

This tells GitHub Pages which gem to use for building:

```ruby
source "https://rubygems.org"

gem "github-pages", group: :jekyll_plugins
```

### Create `.gitignore`

Keep build artifacts out of your repo:

```
_site/
.sass-cache/
.jekyll-cache/
.jekyll-metadata
vendor/
.bundle/
```

### Create the `_posts/` directory

```bash
mkdir _posts
```

Your directory should now look like this:

```
my-org.github.io/
├── .gitignore
├── Gemfile
├── _config.yml
├── _posts/
├── about.md
└── index.md
```

## Step 3: Write Your First Post

Posts live in `_posts/` and must be named with the date-title convention:

```
_posts/YYYY-MM-DD-your-post-title.md
```

For example: `_posts/2026-03-03-hello-world.md`

```markdown
---
layout: post
title: "Hello World: My First Post"
date: 2026-03-03
categories: [general]
tags: [intro]
description: "The obligatory first post."
---

This is my first blog post. It's written in plain markdown,
committed to a git repo, and published automatically by GitHub Pages.

## Why This Setup?

- **Free hosting** — GitHub Pages costs nothing.
- **No server to manage** — push markdown, get a website.
- **Version controlled** — your blog is a git repo with full history.
- **Portable** — it's just markdown files. Move anywhere, anytime.

## Code blocks work out of the box

```python
def hello():
    print("Hello from GitHub Pages!")
```

## So do lists, links, images, and everything else markdown supports.

That's it. Push and publish.
```

### Front matter explained

The section between the `---` lines is called **front matter**. It's YAML metadata that Jekyll uses:

- `layout: post` — use the blog post template
- `title` — the post title (shows up in the page and in the home listing)
- `date` — the publish date (also encoded in the filename, but the front matter value takes priority)
- `categories` and `tags` — optional, useful for organization
- `description` — optional, used by some themes for SEO/meta tags

## Step 4: Commit and Push

```bash
git add -A
git commit -m "Initial Jekyll site with first blog post"
git branch -M main
```

Now create the repo on GitHub and push. If you have `gh` set up:

```bash
gh repo create my-org/my-org.github.io --public --source . --remote origin --push \
  --description "My dev blog"
```

If `gh repo create` fails due to permissions (common with fine-grained PATs on org repos), create the repo manually in the GitHub UI, then:

```bash
git remote add origin https://github.com/my-org/my-org.github.io.git
git push -u origin main
```

**Important:** The repo must be **public** for GitHub Pages to work on the free plan.

## Step 5: Enable GitHub Pages

1. Go to your repo on GitHub → **Settings** → **Pages**
2. Under **Source**, select **"Deploy from a branch"**
3. Set branch to **`main`**, folder to **`/ (root)`**
4. Click **Save**

GitHub will trigger a build. The first build usually takes 1–2 minutes.

### If you see a 404

Don't panic. A few things to check:

- **Build hasn't finished yet.** Wait a minute or two. Check the build status:
  ```bash
  gh api repos/my-org/my-org.github.io/pages --jq '.status'
  ```

- **No build triggered at all.** You can manually trigger one:
  ```bash
  gh api -X POST repos/my-org/my-org.github.io/pages/builds
  ```

- **Org-level Pages policy is disabled.** If you're using an organization, an admin may need to enable Pages under:
  - Organization Settings → Member privileges → Pages creation

## Step 6: Verify

Visit `https://my-org.github.io/`. You should see your blog with the Minima theme, your about page in the header, and your first post listed on the home page.

## Publishing New Posts

From now on, publishing is just:

1. Create a new file in `_posts/` following the `YYYY-MM-DD-title.md` convention
2. Write your content with proper front matter
3. Commit and push

```bash
git add _posts/2026-03-10-my-second-post.md
git commit -m "Add post: My Second Post"
git push
```

GitHub Pages rebuilds automatically on push. Your new post will be live within a minute.

## Customization Ideas (for later)

Once your blog is running, you might want to:

- **Custom domain:** Add a `CNAME` file with your domain (e.g., `blog.example.com`) and configure DNS. [GitHub docs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site).
- **Different theme:** Swap `theme: minima` in `_config.yml` for another [supported theme](https://pages.github.com/themes/), or use `remote_theme` for any Jekyll theme on GitHub.
- **Google Analytics:** Most themes support adding a tracking ID in `_config.yml`.
- **Comments:** Add [Giscus](https://giscus.app/) (GitHub Discussions-based) or [Utterances](https://utteranc.es/) (GitHub Issues-based) for commenting.
- **RSS feed:** The `jekyll-feed` plugin (included in `github-pages` gem) auto-generates `/feed.xml`.

## Quick Reference

| What | Where |
|---|---|
| Site config | `_config.yml` |
| Blog posts | `_posts/YYYY-MM-DD-title.md` |
| Static pages | Root directory (e.g., `about.md`) |
| Build trigger | Automatic on `git push` |
| Manual build | `gh api -X POST repos/ORG/REPO/pages/builds` |
| Check build status | `gh api repos/ORG/REPO/pages --jq '.status'` |

## Summary

The entire setup is:
1. Create a few files (`_config.yml`, `index.md`, `Gemfile`, `.gitignore`)
2. Write a post in `_posts/`
3. Push to a public GitHub repo named `<name>.github.io`
4. Enable Pages in repo settings
5. Wait one minute

No Ruby installation, no build pipeline, no hosting fees. Just markdown and git.
