---
layout: post
title: "Github Is Classy"
date: 2012-03-04 18:42
comments: true
categories: [development, github]
---

Fact: GitHub is classy. This isn't just because
[Scott Chacon](https://github.com/schacon) works there, either. Their
handling of a security issue today was very professional. That said, I
have some words to say about the issue itself and the aftermath, and
things you as an application developer can do to help, and to avoid
this kind of problem.

*Disclaimer: I'm not an application developer. I am a sysadmin with a
 diverse background in operating system security, including previously
 held GSEC and GCUX certifications from the SANS Institute/GIAC.*

*Second disclaimer: This is not an anti-Rails post. All web frameworks
 need to be conscious of security, and take bug reports for security
 issues seriously.*

Issue
=====

*Update - Preface: I am not talking about a security vulnerability (a
la an exploit) in Rails. I am talking about a feature that allows
automatically generated code to do things that are not secure and it
is apparently on purpose and by design. This is the wrong thing to
do. Deny by default with whitelisting is the right thing to do.*

There is a security issue in
[Ruby on Rails](https://github.com/rails/rails/issues/5228). The bug
is closed, but I haven't dug into find out if the actual problem is
fixed.

The bug itself is very serious. It allows a malicious user to send
arbitrary parameters to a Rails application without requiring
whitelisting up front. In fact,
[three months ago an issue](https://github.com/rails/rails/pull/4062)
was opened in the Rails project to force new applications to enforce
whitelist mode by default. That bug was subsequently closed just a few
days ago. I'm not going to do an analysis of the issue itself, you can go
read the linked tickets and do further research on the issue.

*Update* I missed clarifying this. There is a second issue at hand.
 GitHub resolved the mass assignment bug by fixing their application.
 The second issue is that they had a vulnerability in their public key
 form update.

Resolution and Aftermath
------------------------

You can read all about the initial resolution and retrospective on the
[GitHub blog](https://github.com/blog/1068-public-key-security-vulnerability-and-mitigation).
You can also read their
[follow-up post](https://github.com/blog/1069-responsible-disclosure-policy)
on responsible disclosure.

The manner in which they handled the situation is a class act: they
behaved like professionals. Here's why:

* A user reported a problem with their app.
* They worked with the user to resolve the problem.
* The same user exploited another vulnerability to prove a point to
  the Rails project, which is against the GitHub terms of service.
* GitHub suspended the user's account in accordance with their terms
  of service.

As an unauthorized breach of a computer system, what Egor Homakov did
is illegal in the United States. It was also irresponsible and
unprofessional. However, his intent was not malicious. I think GitHub
did the right thing by giving him another chance in reinstating
Mr. Homakov's account several hours after the incident.

GitHub has issued two apologies about this incident. First for the
vulnerability existing in the first place, and second for not being
clear how customers and users can responsibly disclose security
vulnerabilities. They also committed to doing a security audit of
their code base.

GitHub is classy.

Security
========

After the above, I feel compelled to say some more things about
security in general. You are responsible for a lot of things regarding
the web applications in your infrastructure. One of those is security,
and you should do everything you can to write stable, secure code.

In a hackernews thread about this incident, Yehuda Katz said,
"[Not all security vulnerabilities can be protected automatically by a web framework](http://news.ycombinator.com/item?id=3664334)."

That is a fact. However, web frameworks should provide sane, secure
settings by default. Those settings should be modifiable by the
end-user, the developer. If a developer wishes to disable those
controls, they totally have that right. I think that they need to
understand the potential risk that they are accepting, and what impact
that might have on the business/organization implementing the
application.

This is *exactly* like the default setting of Red Hat Enterprise Linux
to enable SELinux by default on new installations. Whether you love or
hate this default, it is sane and secure. System administrators can
then use the system as is, or disable SELinux if that is an acceptable
level of risk.

Clearly in this incident, it is *not* an acceptable level of risk for
GitHub, as they have repaired their application. It would have better
to do that long ago, but at least it's fixed now.

Security and convenience are quite often polar opposites and mutually
exclusive. This is not always the case, but it is true much of the
time, if not most of the time. The choice for Rails to not have
whitelisting by default is in favor of developer convenience. Yes, it
is up to the developer to make their application secure, but that was
already the case. This simply creates extra work for them to do so.

Deny-all by default is the sane, correct and secure posture to take
when building systems. This is the practice of many tools and
operating system defaults - SELinux as mentioned, or "no open ports"
per Ubuntu's practice. You don't have to agree with it, and you
certainly can change it, but that doesn't change the fact that it is
*correct* and *sane*.

Vulnerability Disclosure
------------------------

There is an entire field in information technology devoted to
vulnerability disclosure. This is typically done by people performing
"ethical hacking" and is one thing done when a code base goes through
a security audit. This is a field that has responsible professionals
participating in a variety of companies, and if it sounds interesting
to you, I recommend the variety of courses offered by the SANS
Institute:

* [Web Application Penetration Testing (SEC542)](https://www.sans.org/security-training/web-app-penetration-testing-ethical-hacking-942-mid)
* [Network Penetration Testing (SEC560)](https://www.sans.org/security-training/network-penetration-testing-ethical-hacking-937-mid)
* [Advanced Penetration Testing (SEC660)](https://www.sans.org/security-training/advanced-penetration-testing-exploits-ethical-hacking-1517-mid)

What You Can Do
===============

First of all, understand the security guidelines and best practices
for the programming language you're using. Doing things that are
typesafe, or avoid buffer overflows, that kind of thing. Also
understand and follow the security guideslines and best practices for
the web framework you're using. The Ruby on Rails project has a fairly
detailed
[security guide](http://guides.rubyonrails.org/security.html). If
you're taking shortcuts, understand the possible risks with that. If
you don't know the risks, or understand the guidelines, please ask
someone in the community for help.

I strongly recommend you also learn the security guidelines and
associated best practices for the operating system or distribution
that your application will run on in production. If your organization
has operations staff, I'm sure they can help you learn and understand.
If they don't, they're not doing their job :).

Every organization and every application is different. The security
implications are going to vary by industry. Talk to the business
owners and find out what the level of security risk they are
comfortable accepting.

Above all, be a professional. Don't flippantly close security bugs.
Don't be a dick on discussions about security topics.

In other words, be classy. Like GitHub.

Thank you.
