---
title: "My First Integration Blog post between github and my personal website"
date: "2026-06-07"
tags: ["blog-post", "github", "personal-website", "mermaid"]
excerpt: "How I integrated my github repository with my personal website."
---

# My First Integration Blog post between github and my personal website

Well! This is my first blog where I will note all the trial and error I run into as a Master student. Honestly, writing or keeping notes were never habits of mine. I prefered to keep short notes with just the key concepts and keywords I needed before an exam. I hope this will change that for the better where I could improve on my writing skills and get better at explaining things to others. In the past, I was good at absorbing knowledge but not so good at sharing it.


In this blog post, I will share how I set up my blog so that my writing and my website live in two separate places but still come together as one. The idea is simple. I keep my posts as plain Markdown files in one GitHub repository (`vathapann/lab`), and my personal website is a completely separate project. This way, writing a post feels like writing a note. I don't have to touch the website code at all.

Here is how the two sides connect. My website is built with [Astro](https://astro.build), and I wrote a small custom loader for it. When the site is built, that loader reaches out to the GitHub API, looks inside my `blog-posts` folder, and pulls in every post it finds there. Each folder becomes one blog post, and the folder name turns into the URL of that post. The loader then reads the front matter at the top of each file (the title, date, tags, and a short excerpt) and renders the Markdown body into HTML.

**Diagram showing how the lab repo connects to the Astro website through the GitHub API**

```mermaid
flowchart TD
    user["Visitor"]
    live["`**sovathapann.site/blog** — live blog`"]
    nginx["nginx — serves the static site"]
    list["List page — all posts by date"]
    post["Post pages — one per folder"]
    loader["Astro custom loader — reads front matter, renders HTML"]
    api["GitHub API — queried at build time"]
    repo["lab/blog-posts/ — Markdown posts"]

    user --> live
    live --> nginx
    nginx --> list
    nginx --> post
    list --> loader
    post --> loader
    loader --> api
    api --> repo
```

From there, Astro generates two kinds of pages: a list page that shows all my posts sorted by date, and one page for each individual post. The finished site is served behind nginx on my own server, where it goes live at [sovathapann.site/blog](https://sovathapann.site/blog). The part I like most is the clean separation which I just write plain Markdown in one repo, and my website takes care of turning it into proper, styled blog pages.

And here is the result of my blog page live at sovathapann.site/blog:

![Screenshot of my live blog page at sovathapann.site/blog](https://raw.githubusercontent.com/vathapann/lab/main/blog-posts/my-first-post/assets/sovathapann-site-blog-screenshot.png)
