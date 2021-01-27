---
layout: post
title: "Awesome Syntax Highlighting in Keynote"
date: 2015-02-10 21:31:49 -0700
comments: true
categories: [presentations, utilities]
---

I am working on my presentation for ChefConf. I plan to have quite a lot of code samples. I've found the options for getting code samples with nice syntax highlight a lackluster endeavour, with various GUI editors like TextMate, Sublime, and Atom having "Copy as RTF" plugins, but none of them being easily customizable.

So I did a quick Google search and happened on a gist I hadn't seen before. [It](https://gist.github.com/jimbojsb/1630790) describes the following steps:

1. Install Homebrew (done, I have that covered with Chef ;)).
1. Install "highlight" with `brew install highlight`.
1. Use highlight to transform the source code file to RTF and copy it to the clipboard.
1. Paste the clipboard into Keynote.app.

This isn't much different than the other solutions, except one super cool thing I learned about highlight.

It has styles.

```
highlight -w

<OMG LIST OF EIGHTY TWO DIFFERENT STYLES!!!>
```

That's right, at least at the time of this writing highlight has 82 different styles available. Including my favorite(s) solarized - both light and dark. Note that the `--help` output says that this option is deprecated in the version I've installed (3.18_1), but the styles are in `/usr/local/Cellar/highlight/VERSION/share/themes`.

Highlight knows the syntax highlighting for a lot of languages, these are in `/usr/local/Cellar/highlight/VERSION/share/langDefs`. For example I can get my Ruby recipe highlighted with this:

```
highlight -s solarized-light -O rtf recipes/client.rb | pbcopy
```

This will use Courier New as the font, and depending on the theme/style used, some of the highlighting may be bold or italic. This is easy enough to change in Keynote though.

For up to date documentation and information about highlight, visit the [author's page](http://www.andre-simon.de/doku/highlight/en/highlight.php).
