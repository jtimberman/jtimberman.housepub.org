---
layout: post
title: "Blog Moved to GitHub Pages"
date: 2011-09-29 22:59
comments: true
categories: [development, personal]
---

I moved my blog from Posterous to GitHub Pages. Posterous isn't a bad
system and service. It just didn't fit the way I wanted to manage my
site and the content. It was entirely adequate to get going but I was
dissatisfied with the web form for creating new posts. It also loaded
pages fairly slow.

## GitHub is awesome.

Many others have migrated their technical blogs from services like
Posterous, Wordpress, Tumblr, Blogger and so on over to GitHub
Pages. Most often, this is because these services are database backed
systems, use a web form for creating posts, and don't have a revision
control system that we all know and love under the covers.

[GitHub Pages](http://pages.github.com) is a great service by GitHub
that allows you to serve up static content out of a Git
repository. You can do simple static HTML pages, or you can use a
static page generation framework to generate the content out of
Markdown (or Textile, or HAML) files. There's a number of options for
this, such as [Nanoc](http://nanoc.stoneship.org/) or [Jekyll](http://jekyllrb.com/).

## I chose Jekyll.

_"Jekyll is a blog-aware, static site generator in Ruby"_ -
[http://jekyllrb.com/](http://jekyllrb.com)

Jekyll fits the bill for me. It supports different markup languages,
such as Markdown or Textile, and GitHub Pages has easy instructions on
how to get going with it. Jekyll also includes a *fantastic* Posterous
importer, which flawlessly imported all my
[old posts](/blog/archives). Now I can start creating new posts as
Markdown in Emacs, instead of using a web form, or editing HTML
directly. I can also save revisions using Git, instead of saving a
draft and monkeying with an interface to recover drafts or publish
them.

I sat down a few days ago and imported my old posts. Then I ran
through trying to get them to look alright with a decent CSS
theme. However, I'm not a front end designer/developer, and I never
remember how do do anything useful with CSS. So I went looking for
canned ways to build a nice interface/theme for this blog.

I remembered
[Twitter's Bootstrap](http://github.com/twitter/bootstrap) project,
and thought that might be interesting. After looking at it I was
confused about what to use, since it is a very complete solution,
including iOS mobile layouts in addition to normal web pages. After
looking elsewhere, I saw mention of [Octopress](http://octopress.org),
a framework built on top of Jekyll. I queried
[some](http://junglist.gen.nz) [people](http://likens.us/) that had
experience with it, found that it would probably meet my needs, and
gave it a whirl.

Naturally if you're familiar with Octopress, you can see the result of
that evaluation. It is a really nice system for managing the
site.

## I like my blogging workflow.

I like my new blogging workflow. It is very natural to me as a system
engineer, as it uses Emacs, Rake, a Web Service, and Git. I'll walk
through the steps I took in creating this post. I'm going to skip the
setup of Octopress, as that is
[documented elsewhere](http://octopress.org/docs/setup/).

    % rake new_post'[Blog Moved to GitHub Pages]'
    Creating new post: source/_posts/2011-09-29-blog-moved-to-github-pages.markdown

Then I load up the source file created by the rake task and enter all
this content you see, saving my work along the way. I did, but could
choose, to commit to the git repository as well. In order to preview
the content, I can actually run the entire blog with Octopress's built
in rake task:

    % rake preview
    Starting to watch source with Jekyll and Compass. Starting Rack on port 4000
    [2011-09-29 23:06:20] INFO  WEBrick 1.3.1
    [2011-09-29 23:06:20] INFO  ruby 1.9.2 (2011-07-09) [x86_64-darwin11.1.0]
    [2011-09-29 23:06:20] INFO  WEBrick::HTTPServer#start: pid=92373 port=4000
    Configuration from /Users/jtimberman/Development/jtimberman.github.com/_config.yml
    Auto-regenerating enabled: source -> public
    [2011-09-29 23:06:21] regeneration: 107 files changed
    >>> Compass is watching for changes. Press Ctrl-C to Stop.
    127.0.0.1 - - [29/Sep/2011 23:06:24] "GET / HTTP/1.1" 200 50231 0.0181

Now I can browse to [http://localhost:4000](http://localhost:4000) and see exactly what the
blog post will look like, and it loads lightning fast. This is of
paramount importance for me, as I travel a lot and would really like
to develop blog entries while I'm on airplanes with no internet
access.

Once I am satisfied with the content and how it looks, I commit to the
repository and push the source branch.

    % git add source/_posts
    % git commit -m 'Blog moved to github pages'

Then, it is time to deploy the blog to GitHub Pages. Octopress makes
this easy with another rake task:

    % rake deploy

This takes care of all the repository work required, and pushes to
GitHub. Then I push the actual source branch.

    % git push origin source

## Nice things about Octopress

Besides the rake tasks and that works with Jekyll, Octopress has some
really nice features.

First of all, the default theme is really nice. One thing I wanted out
of CSS on top of Jekyll was something large enough that I could read
easily. I think this theme looks pretty good and is quite readable
with my default browser settings. It is also nice and easily readable
on my iPhone.

Second, Octopress is very customizable. I can easily modify the layout
by editing the `_config.yml` file. Of course I can change the theme,
but I can add JavaScript, layouts and more. I modified the footer to
include my Creative Commons license for the content of this site.

Third, I really like the default layout in this theme. The recent
posts, GitHub repo links and my last few tweets are a nice
touch. Plus I have integrated Disqus for comments and set up Google
Analytics easily, thanks to the ease of customizing the configuration.

## Conclusion

In closing, I'm very happy with this solution. I now have a blogging
workflow that I'm comfortable with, and that will hopefully result in
more posts.
