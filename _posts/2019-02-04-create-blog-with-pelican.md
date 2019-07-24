---
layout: post
title: "Creating a Blog using Pelican"
categories: [Engineering]
tags: [python, pelican, blog]
---

I am not a person, who can do nothing for very long. So, when I found myself in
between university and my first developer job, I decided on starting a Blog.
This is it. Welcome :-)

Before starting the blog, I had some requirements for it:

1. I wanted to host it myself.
2. I wanted to create it using `Python`.
3. Publishing should be as easy as possible.
4. I wanted to write articles on my own machine and publish them by simply
5. uploading them to the server.
6. I wanted to write articles in the [Markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) format.

The Python community being the Python community, I immediately found an
interesting blogging framework called [Pelican](https://blog.getpelican.com/).
It offered static file generation from Markdown files, plenty of plugins and
themes, and with ~8.5k start on GitHub it seemed to be pretty battle tested.
Here is how I set up Pelican on my CentOS 7 server.

## Install Pelican

Pelican own [installation instructions](https://docs.getpelican.com/en/stable/install.html)
are pretty explicit already, but let me walk you through them step-by-step.

Before getting started, make sure to create a new
[Virtual Environment](https://docs.python-guide.org/dev/virtualenvs/#lower-level-virtualenv)
for `Pelican`. Then, execute the following commands one-by-one.

```shell
# Install Pelican, Markdown, and all other dependencies
pip install pelican markdown

# Create a directory for your blog
mkdir blog

# Move to the new directory
cd blog

# Create the Pelican Project
pelican-quickstart

# Answer all the questions. You can later on always
# change your answers in the generated pelicanconf.py file
```

## Writing content

You can use Pelicans [documentation](https://docs.getpelican.com/en/stable/content.html)
to explore all ways of writing content, but for an initial setup, create a
sample post like this:

```shell
# Create a folder that will hold all of your posts
mkdir content

# Create a first sample post
touch content/sample-post.md
```

Fill the `sample-post.md` file with the following data:

```
Title: My First Post
Date: 2019-02-04
Category: Test

This is my first post. Here are some tests:
This is a test of **bold text**
This is a test of *cursive text*
This is a test of a [link](https:/www.google.com)
This is a test of some code: `python main.py`
```

## Publishing content

After you created a first post, run `pelican content` to generate the static
`html` pages from your posts in `content`. To view your blog, run
`pelican --listen` and navigate your browser to [http://localhost:8000](http://localhost:8000).

If you want auto generation of pages whenever you add or modify a post in
`content`, open another tab and run `pelican -r content`. Pelican will now
listen for any changes to the `content` folder and automatically re-run
`pelican content` whenever something changes.

Alternatively to running `pelican -r content`, you can also either install
`Invoke` with `pip install invoke` and run `invoke regenerate` or simply use
your built-in `make` command and run `make regenerate`. Both commands will work
just as `pelican -r content`.

## Conclusion

Congratjuliations! You have just created your first blog! If you want to
customize your blog further, check out Pelicans [Themes](http://www.pelicanthemes.com/)
and [Plugins](https://github.com/getpelican/pelican-plugins).
