---
layout: post
---

> but if i were someone who wanted to put some malicious code into a LOT
> of rails projects this would be an opportune time to phish the authors
> of the bootstrap-sass gem
>
> -- me on Slack, March 26, 2019 at 3:45PM

A little more than a week ago [I found a
backdoor](https://github.com/twbs/bootstrap-sass/issues/1195) in the
`bootstrap-sass` gem. Liran Tal from Synk [wrote a nice
overview](https://snyk.io/blog/malicious-remote-code-execution-backdoor-discovered-in-the-popular-bootstrap-sass-ruby-gem/)
of the timeline of the vulnerability and the technical details involved.

I am not a security expert; however, as this situation shows, software
engineers should have at least passing a "security sensibility" in order
to do their jobs. I've seen comments asking, "did we get lucky"?  I'm
not sure it's luck, unless you're talking about our project using this
particular version of this particular library. I hope that if it was
someone else who encountered these circumstances, they would've acted
similarly.

In any case, here's what happened: One of the builds on an internal tool
we're developing failed unexpectedly. This is not great, but also not
unusual. We saw that the version of `bootstrap-sass` the application
depended on (3.2.0.2) wasn't downloading from RubyGems. I checked and
noticed it was yanked. Again, this isn't unusual: stuff gets yanked from
time to time. I saw that a newer version of the gem was released
(3.2.0.3). I figured some sort of vulnerability was found or something,
and was about to upgrade our `Gemfile` except

* I couldn't find 3.2.0.3 in the GitHub repo
* I couldn't find any mention of the version in any issues or changelogs
* I couldn't find any mention of the gem being yanked by the owners of
  the project

This last part bothered me the most, and that's when I started to get
suspicious. Why would the authors yank a gem and not mention it? Why
would they publish a new version and also not mention it? You would
think there'd be some sort of social trace if there were some sort of
weird security or copyright issue or something, but I could find
nothing.

So I downloaded the gem and poked around it; at this point I was just
trying to figure out what the change was. There _was_ a changelog entry
in the gem, stating only `Recompile with libsass`. And yes, it's true
the ruby `sass` gem is now deprecated in favor of `libsass`, but that
didn't seem like a particularly great reason to yank a dependency and
release such a huge change as a patch release. It was at this point that
my curiosity escalated to alarm, leading to the Slack message
introducing this post.

Thankfully `bootstrap-sass` is a very small gem, and I found the
offending code pretty quickly with a crude and manual "diff". The red
herring of the CloudFlare-style cookie threw me off for a second; I was
caught up in the the fact that the cookie had a key of `___cfduid`, but
it didn't occur to me at the time that this was an intentional
obfuscation. Regardless, I both filed the issue on GitHub as well as
alerted RubyGems that something fishy was going on.  The community was
pretty quick to take action where they needed to, and everything was
back to normal and surfaced as well as it could be (it looks like some
auditing improvements have been filed with the RubyGems team).

One funny thing is that, when I look at the issue I created, in
retrospect, I see that I didn't call what's clearly a backdoor exactly
that. I think this is partially because I stopped analyzing once I
realized something was strange, and tried to escalate things as quickly
as possible, and partially because _I couldn't believe_ that such a
popular gem had a backdoor in it.

Overall, I think this situation shows the importance of layered
security. The attackers here didn't gain access to the authors' GitHub
or Twitter accounts, which made the attack more suspicious and therefore
that much weaker. This wasn't an accidental or "lucky" situation--the
circumstances around it were suspicious enough to warrant me auditing
the code, which lead to the discovery. I don't think this would prevent
a more subtle attack (like an "inside job" situation), nor does it
address the trend where knowing what's in a project's dependency tree is
increasingly intractable, but maybe if we all spend the half-hour
investigating something suspicious from our own perspective, we can all
sleep (a little) easier.

Thanks to [Mike Kendall](https://github.com/zenkalia) for looking at a
draft of this post.
