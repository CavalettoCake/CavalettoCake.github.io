---
title: "GitHub Shortcuts"
date: 2023-12-01 13:45:00 -0500
categories: [Varia]
tags: [github]
render_with_liquid: false
---

A few quick reminders on using [GitHub Pages](https://pages.github.com/) to host a blog such as this one.  I am using the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme using the [Jekyll tool](https://jekyllrb.com/). 

## Guides

1. Head over to [GitHub](https://github.com/), a web-based version control service, and create an account.
2. Install [Git](https://git-scm.com/), a version control system that works on your computer.
3. Install Jekyll, a static site generator, following their [installation guides](https://jekyllrb.com/docs/installation/).
4. Head over to Chirpy's [Getting Started](https://chirpy.cotes.page/posts/getting-started/] guide and follow it.
  + Choose the fork option. Otherwise, you will have more files to create to get your build to deploy correctly
  + Even with the fork option I met roadblocks due to missing configurations (on Windows). I had to run an extra command line found on Stackoverflow [here](https://stackoverflow.com/questions/72331753/ruby-and-rails-github-action-exit-code-16)

## Useful commands

You can edit your files on GitHub and commit every change, but is more efficient to do so locally and commit once you are all done. Here are some useful commands. They must all be **run from your site’s directory**.

### Git commands

See changes about to be staged
: `git status`

Stage changes
: `git add FILENAME` or `git add .` to stage all changes.

Commit

: `git commit -m "MESSAGE"` to commit changes to the branch.

Update the GitHub repository with your local repository
: `git push origin master` when `master` is the name of you site's branch. Replace with whatever it is called.

### Ruby commands

Before pushing your repository you might want to check what the changes look like on your site. This command starts a development server on your local machine and allows you to view your Jekyll site in a web browser.

Run the Jekyll server
: `bundle exec jekyll serve`

The command will provide a URL that looks like this: *http://localhost:4000*. Copy-paste this in a browser and you will access your site offline.
