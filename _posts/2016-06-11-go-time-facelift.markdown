---
layout: post
title:  "Go Time update"
date:   2016-06-11 00:00:00
categories: golang time gotime web tool
---

I've made some improvements to [Go Time](http://gotime.agardner.me/):

- A minor facelift. Since apparently [ugly websites are cool](https://www.washingtonpost.com/news/the-intersect/wp/2016/05/09/the-hottest-trend-in-web-design-is-intentionally-ugly-unusable-sites/), I had to step up my CSS game. Who doesn't love a fixed header?

- New pattern testing functionality! Now you can test your input strings against Go time patterns and see the output. This is all implemented client-side in Javascript, so it's a bit faster than the Go Playground.

- The entire site is now open-source, and the code for parsing patterns has been broken out into a separate file. There are a couple unit tests and scant documentation at the moment. The repo is [here](https://github.com/alanctgardner/gotime).

The actual quality of the code is still not good. I'm not really a Javascript expert, and I haven't been very rigorous about building out this little tool. Hopefully the code can be of use to somebody.
