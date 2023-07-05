---
layout: post
title: "Setting up the blog 📔"
date: "2023-03-26"
---

## Attempt #1: Using a template repo

For my first attempt to set up this blog, I used the instructions in [this article](https://chadbaldwin.net/2021/03/14/how-to-build-a-sql-blog.html).

The instructions and template repo made the process very simple and straight-forward, and much less daunting than some other potential approaches.

Using those resources, I was able to initialise the blog site in less than 5 minutes, which allowed me to spend more time exploring the config and writing this post.

### Step 1 - Initialising the blog 🛠️

The process was incredibly straight-forward: I simply had to generate the blog repo via [this link](https://github.com/chadbaldwin/simple-blog-bootstrap/generate), which I had to name `<my-username>.github.io` in order to be recognised as a "GitHub Pages" repo.

That was it!

At this stage, GitHub had immediately started hosting the blog site, with all of the placeholder details from the repo, and it had taken less than 5 minutes to do.

### Step 2 - Updating some placeholder details 📃

Next, I was ready to start replacing any placeholder details with my own. This included:

- Adding a very basic README file
- Updating my email and social media handles
- Updating the blog site title
- Updating the blog site description
- Updating the blog site index/intro text
- Writing the first blog post (this one! 😄)

### Pausing to reflect on attempt #1

Whilst this initial version of the blog was nice, it left me with a lot of questions, mainly:
- How do I test this locally?
- How do I customise some of the HTML and CSS myself?

## Attempt #2: Using the GitHub Docs

My next attempt used the instructions in GitHub's docs for this: [Creating a GitHub Pages site with Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll).

### Pros of this attempt ➕

A massive plus of this approach was that I was able to almost immediately start testing the repository locally, using `bundle exec jekyll serve`, which hosted the pages locally on `http://localhost:4000` for me to be able to manually test how any changes I made would impact the site.

Another very exciting feature of this is that the Jekyll local server reloads whenever I save any updates to any of the files (excluding the `_config.yml` file), meaning that I could save a change, refresh the local webpage, and see my changes immediately.

### Cons of this attempt ➖

This attempt was a lot slower, as it required me to understand a lot of more what was going on under the hood (at least, as much as I could learn without having to dive into the Jekyll source code).

## Conclusion

Overall, I'm really happy with my second attempt at getting the blog up and running.

I'll definitely return to do some further enhancements to the blog later on, including copying across some of the features from attempt #1, such as the sharing links and the post navigation links.

I may also consider some other enhancements, such as adding some automated testing, and running the tests in CI (via GitHub Actions).

I still haven't learnt how to override the Jekyll theme's HTML layouts and CSS, I'm hoping to learn this next and to make some tweaks to the current versions.

For now, I'm very excited to have my own blog up and running! 🥳🎉
