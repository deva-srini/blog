# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog (Deva's ML Notes) covering ML, LLMs, and AI. Built with Jekyll using the Minima theme, hosted on GitHub Pages.

## Development Commands

```bash
# Install dependencies
bundle install

# Serve locally (http://localhost:4000/blog/)
bundle exec jekyll serve

# Build without serving
bundle exec jekyll build
```

Requires Ruby with Bundler. The `webrick` gem is included for Ruby 3+ compatibility.

## Architecture

- **Static site generator**: Jekyll with `github-pages` gem (pins Jekyll version and plugins to match GitHub Pages)
- **Theme**: Minima (installed via gem, no local overrides yet)
- **Markdown engine**: kramdown with Rouge syntax highlighting
- **Plugins**: jekyll-feed (RSS), jekyll-seo-tag
- **Permalink pattern**: `/:year/:month/:day/:title/`
- **Base URL**: `/blog` (site lives at `deva-srini.github.io/blog`)

## Content

- Posts go in `_posts/` with filename format `YYYY-MM-DD-title.md`
- Post front matter requires: `layout: post`, `title`, `date`, `categories`
- `show_excerpts: true` displays post excerpts on the homepage
- `about.md` and `index.md` are standalone pages

## Key Config

`_config.yml` excludes `Gemfile`, `Gemfile.lock`, and `README.md` from the built site. The `_site/` directory is the build output (gitignored).
