---
layout: post
title:  "Go Time - An online Golang date format tool"
date:   2015-10-16 00:00:00
categories: golang time format go date 
---

[Go Time - an online Golang date format tool](http://gotime.agardner.me)

I'm often frustrated by Go's syntax for date-time parsing. [The docs](https://golang.org/pkg/time/#pkg-constants) explain the concept: you write the reference date (January 2nd, 2006 at 3:04:05PM in the MST time zone) in the format you want to parse or print. They provide a few examples, but the only complete list of format parts is in the time package codebase. It's not really user-friendly.

To make things easier, I wrote a quick webapp that parses Go date-time formats and explains the components. The page also includes a complete list of all the different formats for each part of the string, with explanation. [Try it here](http://gotime.agardner.me).

Technologically, there's no Go involved in this project - the page uses a line-for-line Javascript reimplementation of the Go format-string parser to produce the explanations. The list of symbols and explanations were also taken directly from [format.go](https://golang.org/src/time/format.go). In the future I'm planning to port the parser responsible for applying the patterns, so we can have in-browser testing of patterns against example strings. 
