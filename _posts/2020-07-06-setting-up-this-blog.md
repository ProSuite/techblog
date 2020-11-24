---
layout: post
title:  "Setting up this Blog"
author: ujr
date:   2020-07-06 20:00:00 +0200
---

This article explains the setup of this blog.
The two essential technical ingredients are
the [Jekyll][] static blog generator and the
free [GitHub Pages][ghpages] hosting service.

We found it wishful to have a blog for writing about
technical topics related to our everyday work and in
particular related to ProSuite development.
The articles here are expected to be of most value to
ourselves, but if they find a wider use that is much
appreciated.

## About GitHub Pages

- a **static site** hosting service (no server-side code)
- publishes any static files straight from a repository,
  optionally run through a builder (Jekyll by default)
- having domain `user.github.io/repository` (other possibilities exist)
- GitHub Pages are **public** even if generated from a non-public repo!

Three types of sites:

1. Project (has a few options for the publishing source)
2. User (always from branch `master` in repo `user.github.io`)
3. Organization (always from branch `master` in repo `org.github.io`)

This blog was initially set up as a project site, using the `master` branch
of the [blog's repository][blogrepo] as the publishing source.

Usage limits (as of July 2020, will hardly hurt us):

- data: max 1 GB (soft for source repo, hard for published pages)
- bandwidth: 100GB per month
- building: 10 builds per hour

## About Jekyll

A blog-aware static site generator written in Ruby:

- supports [Markdown][gfm], [Liquid][liquid] (templating)
  and [Sass][sass] (CSS preprocessor)
- configuration: *_config.yml* in root dir (should rarely change)
- Jekyll does not build files/folders
  starting with `_`, `.`, `#` or ending with `~` (can override in config)
- “Front matter” to set or override title, layout, permalink, category, etc.
- Themes: available, start with default theme,
  can easily customize or write your own

## Installation

Local installation assuming Windows (each contributor, optional):

- Install Ruby: use the RubyInstaller from <https://www.ruby-lang.org/en/documentation/installation/>
- Optional: Install Bundler: see <https://bundler.io> (I did not do it)
- Install Jekyll: see <https://jekyllrb.com/docs/installation/windows/>

The remaining steps are about creating the repository
and the initial content. Do not repeat -- it was already done
and is documented here only for reference.

- Log in to <https://GitHub.com/ProSuite>
- Create new repository `techblog`, visibility `Public`
- Open a Git Bash command line
- `git init ProSuiteTechBlog`
- `cd ProSuiteTechBlog`
- `jekyll new .` (creates the directory structure and a sample post)
- optional: edit partials, styles, posts, etc.
- edit `_config.yml`: set `baseurl: /techblog` and `url: "https://prosuite.github.io"`
- `git commit -m "Initial commit"`
- `git remote add origin https://github.com/ProSuite/techblog.git`
- `git push -u origin master`
- Go again to <https://GitHub.com/ProSuite/techblog>
  and in the repo's settings, choose “master branch”
  for the source; do not modify the theme.
- Now GitHub Pages should build the site; this may take some time.
- Check the generated site: <https://ProSuite.GitHub.io/techblog>

## Authoring

Git clone this repository.  
Follow instructions in the root README.md file.  
You need Jekyll and Ruby only for previewing.  
If bold enough, just write your articles.

[Jekyll]: https://jekyllrb.com
[ghpages]: https://pages.github.com
[blogrepo]: https://github.com/ProSuite/techblog
[gfm]: https://guides.github.com/features/mastering-markdown/
[liquid]: https://shopify.github.io/liquid/
[sass]: https://sass-lang.com/
