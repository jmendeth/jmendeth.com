---
layout: post
title: "SnowShoe Stamp: first impressions"
cover: no
#  image: /media/2014-08-25-snowshoestamp/cover.png
quote: Some thoughts about these ingenuous tags.
comments: 2014-08-25-snowshoestamp
---

Today I saw [SnowShoe Stamp](http://snowshoestamp.com).

SnowShoe Stamp describes itself as a great way to link physical assets to
virtual ones. I have to admit it: when I heard of them, my first thought was
"Oh no, yet another company that rebrands NFC".

Later I realized I was completely wrong.

SnowShoe is about stamps that are, well, stamped onto a touchscreen, and then
recognized by the device.

<iframe width="560" height="315" src="//www.youtube.com/embed/eoTPAoKFKDo" frameborder="0" allowfullscreen></iframe>

The idea behind SnowShoe Stamp is simple: the stamp touches the screen and
produces touches that allow the app to detect and identify it. No RF involved,
just a touchscreen.

I *absolutely love* the idea. 

So I went to [their Github](https://github.com/snowshoestamp) to learn more
about the software side (aka SDK) of their engine. Here's what I found out.

> Note: this post is a draft. There may be some typos or wrong assumptions,
> because I didn't spend as much time verifying the info as I usually do.

> **Edit:** they finally released a JS SDK that eliminates the need for a stamp
> screen, but the other issues explained here still apply.


## Code

Here's the repo that allows one to "host a stamp screen": [Touchy SDK](https://github.com/snowshoestamp/touchy_sdk)

When the user stamps in this webpage, the data is sent to a "Callback URL".
To use this stamp screen, you copy the files and edit the `index.html` file to
insert your own callback / error URLs and background.

The idea of having to direct your user to a specific webpage, stamp, and then
go back to your application doesn't feel right at all, but I supposed the logic
was too complex to be embedded in another webpage.

But it got worse.

Trashy code. Poorly formatted. Hard to understand. Almost no comments. Didn't 
follow good practices. I had to carefully read it a few times to know what each
variable was used for.

The important code is in `index.html`. It uses [Touchy.js](http://touchyjs.org)
to wait for five simultaneous touches to be active, grabs the coordinates of
each of them, puts them in an object, and sends it as Base64-encoded JSON to
the Callback URL.

If anything other than five touches is detected for more than two seconds, or
detected more than two times, the user is redirected to an "Error URL".

**It could have been better. Way better.**

I understand the developers are better at Ruby or Python than at Javascript,
but the stamp screen is the core of the project and deserved a better
implementation. It totally did.

But the client-side logic is so simple that I could implement it myself in
minutes, so I don't care so much about the code.


## Design

This is the real problem. I was expecting an SDK that allowed me to match
the points to the identity. Instead, I have to pass the points to *their* API
to do the matching.

I can't work offline. I can't do it locally. No, I have to wait for the request
to go to my server, my server makes an OAuth request to their API, their API
returns the response, my server acts upon that and the client receives the
feedback.

I wanted a company that would give me the hardware part (the stamps) and would
leave the software part to me, as happens with NFC tags, or the good ol'
[fiducials](https://en.wikipedia.org/wiki/Fiducial_marker)! I didn't want to
register an API key and hand out every request to them. And what if they suddenly
decide to start charging for their API?

That's not an ideal business model. SnowShoe: You're doing it wrong.


## It's magic

SnowShoe markets their product like magic. They literally do.

> SnowShoe provides software that once integrated into your project, can identify the stamps when touched to the screen of a multi-touch device, allowing the magic to ensue.

Anywhere in their website is actually explained that stamps produce five touches at
different positions, and the coordinates are sent to the server for matching.
It's magic.

They never talk about touch coordinates or encoded JSON, they say "pattern",
"digital identity" and "stamp observation data". It's magic.

*Even the "How it Works" page is magic*, they mention the `data` parameter, but
<del>never make an indication to what is actually in that data</del>.  
**Edit:** they do, they say "points" are being captured.

I'm a developer, not a kid.

But here comes the worst part: **it's secret**.

> SnowShoe makes individual pieces of plastic, where each one carries a different *secret digital identity*.

> They are *nearly impossible to spoof*.

So. You're saying five bidimensional points are impossible to spoof? Wow.
I'm pretty confident that a stamp that has no moving parts, no electricity and
no actual interaction with the smartphone is vulnerable to [replay 
attacks](https://en.wikipedia.org/wiki/Replay_attack).

So yeah, it's impossible to spoof until it's stamped. Then its identity is no
longer secret or unique. Again, magic.


## Conclusion

These stamps are a great idea, but I was really disappointed to see how the idea
is being implemented. This isn't good for anyone.

Don't get me wrong, I love what these guys are doing. But it makes me sad that
the project is taking that direction. If I could work with the company to make
a client-side engine from scratch, I'd do it without questioning.

If SnowShoe opens their magic to the public; if they let the people know how the
stamps actually work, great ideas could be implemented, like:

 * Detecting the position of the stamp, to make the user stamp in different
   spots of the screen.

 * Detecting the time a stamp is in the screen. Longer stamps give different
   feedback or unlock hidden features.

 * Detecting the rotation of the stamp. The user could press and rotate the
   stamp as if it was a key.

Innovation like this would give SnowShoe the push it needs to become a success.
