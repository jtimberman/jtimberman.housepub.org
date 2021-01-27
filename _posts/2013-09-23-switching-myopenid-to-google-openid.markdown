---
layout: post
title: "Switching MyOpenID to Google OpenID"
date: 2013-09-23 10:38
comments: true
categories: [openid, authentication, security]
---

You may be aware that MyOpenID is
[shutting down in February 2014](http://thenextweb.com/insider/2013/09/04/myopenid-to-shut-down/).

The next best thing to use IMO, is Google's OpenID, since they have
2-factor authentication. Google doesn't really expose the OpenID URL
in a way that makes it as easy to use as "username.myopenid.com."
Fortunately, it's relatively simple to add to a custom domain hosted
by, for example, [GitHub pages](http://pages.github.com/). My
coworker, Stephen Delano, pointed me to this pro-tip.

The requirement is to put a `<link>` tag in the HTML header of the
site. It should look like this:

```html
<link rel="openid2.provider" href="https://www.google.com/accounts/o8/ud?source=profiles" />
<link rel="openid2.local_id" href="http://www.google.com/profiles/A_UNIQUE_GOOGLE_PROFILE_ID />
```

Obviously you need a Google Profile, but anyone interested in doing
this probably has a Google+ account for Google Hangouts anyway :).

If you're like me and have your custom domain hosted as an
[Octopress](http://octopress.org/) blog, this goes in
`source/_includes/custom/head.html`. Then deploy the site and in a few
moments you'll be able to start using your site as an OpenID.
