---
layout: post
title:  Office Bet
---

Before leaving on vacation, my friend and coworker at
[Kissmetrics](http://www.kissmetrics.com), [Brett](https://github.com/bhardin),
offered $5.oo to anyone who could correctly guess the number of emails that he'd
have in his inbox upon returning ten days later.  He would use Price-is-Right
rules, so whoever guessed closest without going over would win.  Most people on
the team guessed around 100 emails, with a few cutthroat engineers betting one
email over their friends.

To be safe, I bided my time, and in the end submitted the highest guess of
2,ooo emails.  That would mean roughly nine emails an hour for the next ten days...

I immediately set to work, and it was with great satisfaction that I saw a
smiling Brett return only a few minutes later to declare that his inbox had been
flooded with +2,ooo emails.  Looks like I win???

I may have cheated, but it was all in good fun, and better still, I didn't need
to write a line of code or spend any money to do it.

Spam your Friends!
---

My first thought was to schedule a recurring email.  For this, I turned to
[NudgeMail](http://nudgemail.com/), which my friend
[Brantley](https://github.com/bbeaird) had helped build.  NudgeMail is a free
service that parses emails sent to their `@nudgemail.com` domain.  They look for
a subject line specifying when they should send you a reminder email, called a
"nudge".  It's a great service, but I was definitely trying to abuse it here.  I
tried specifying `every5m` and combinations thereof, but it appears that
NudgeMail is hip to this sort of thing.  Eventually, I settled for setting
numerous `hourly` reminders instead.

One crucial consideration was to avoid sending emails that Brett could
unsubscribe from, so I had the nudges sent to my work email, where a gmail
filter I set up would forward them along to him.  Brett can't really block my
work email, as I could potentially send him something important.  While this
looked promising, I couldn't help but want something bigger.

```
me --> nudgemail

nudgemail -hourly-> my filter -hourly-> brett
```

I immediately thought back to the Citizen Developer mentality my good friend
[Sherif](https://github.com/amgando) tried to instill in me, which emphasized
prototyping rapidly without writing any code at all, primarily using SaaS to
accomplish what would inevitably take longer to code from scratch anyway.  One
service he had introduced me to was [Zapier](https://zapier.com).  Zapier is
like [IFFT](https://ifttt.com/), and it allows you to create actions for
particular triggers.  It acts as glue between various APIs.

I setup a webhook-to-email zap, so that any hit to a given URL would send me an
email, and then I'd let the gmail filter forward the email along to Brett.

```
visit URL --> triggers email --> my filter --> brett
```

The trick now was to trigger the endpoint *en masse*.  To
do so, I created a facebook note.  Facebook notes allow you to include HTML, and
crucially, they don't cache images with distinct query-string parameters.  This
means that when facebook encounters the HTML for a remote image:

```
<img src="https://www.images.com/some-image.jpg" />
```

It visits the source to load the image.  Normally, requests to the same resource
would be cached to prevent making several requests to that resource, but the
facebook scraper considers resources with different query-string params to be
distinct, and it *does not* cache them.  That means the following will trigger
one request to the given resource for each image:

```
<img src="https://zapier.com/hooks/catch/.../?parameter=1" />
<img src="https://zapier.com/hooks/catch/.../?parameter=2" />
<img src="https://zapier.com/hooks/catch/.../?parameter=3" />
...
```

[This is a known bug](http://chr13.com/2014/04/20/using-facebook-notes-to-ddos-any-website/).
Thus, everytime someone views the note, hundreds of requests would be sent
against the Zapier webhook.  Moreover, the unique parameter can be captured by
Zapier and passed into the email, which makes each email unique, so gmail's spam
blocker ignores them.

```
View note --> load images -x100-> URL -x100-> email -x100-> brett
```

To draft the hundreds of "images", I just used the following in `irb`:

```
0.upto(100).each do |num|
  p "<img src='https://zapier.com/hooks/.../?parameter=#{num}' />"
end
```

and relied on my text editor to strip the `"` from each line.

In the end, we ignored my emails and played the game legitimately, but this was
still good for a laugh or two(thousand).
