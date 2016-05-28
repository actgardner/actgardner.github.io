---
layout: post
title:  "Having a bad time with Golang"
date:   2016-05-27 00:00:00
categories: golang time design
---

At my day job we ship a product that (in part) parses log messages. We use Go for log collection and parsing, and generally this works pretty well - there's another blog post in the works about why Go is great for this use case. This post is about something Go does not do well. It does it _weirdly_. There are a few weird things in Go: 

**Regexes?** RE2, not PCRE. 

**Date formats?** Seemingly made up on a whim.

For a full-time Go developer these are pretty minor adjustments. You learn the language, you figure it out, it's a little annoying. In this case, it's different.

I need to give our Field team and our customers a flexible, configurable tool for log management. Flexibility means exposing some implementation details, like regexes and date formats. Our Field team is acutely, painfully aware of how Go handles regexes and time formats. I wrote [Go Time](http://gotime.agardner.me) as a tool for them, because the format was new and unusual. I've since discovered [Flipping Go Date Format](http://flippinggodateformat.com/), which is much prettier if you've already memorized `strftime` formats. The weird format is tough, but we can work around it and be successful with tooling and documentation.

Something we can't fix with tooling and docs is [this three year-old bug](https://github.com/golang/go/issues/6189) to support commas as a delimiter for fractional seconds. In other words, this is a time you can parse:

```
21:00:57.012
```

but this is a time you **cannot** parse:

```
21:00:57,012
```

You could, in theory, leave off the fractional seconds in that example and parse the rest of the date. How do you handle time zones?

```
21:00:57,012+06
```

You can't.

I don't feel like it's reasonable to punt on supporting commas as delimiters for _three years_. I definitely couldn't tell our customers and Field to wait for three years. We do log management, and a lot of customers use Log4j. Log4j uses the standard ISO8601 date format. These are not unreasonable things to ask your log management vendor for. 

In an ideal world, I'd love to rewrite the Go time parser to use a different syntax alltogether. I'd construct a `time.Time` struct using all the fields I had parsed out. The clouds would part, a chorus of angels would sing, and Field and customers would give me high fives as we all drove our solid gold cars into the sunset.

In real life, engineering is about compromise. How much time can I spend solving this problem? How much time _should_ I spend? It's painful, but can I fix 80% of the pain in 20% of the time? How about 2% of the time? It turns out there's a really simple solution:

_type punning_

That bug I linked above? I wrote a PR to fix it. It wasn't very time consuming. It also didn't get merged. 

I can't just keep the code in my repo: the parser requires access to private members of the `time.Time` struct. I can make my own fork of the `time` package, but then the struct won't be a `time.Time`, it'll be a `github.com/alanctgardner/allthebadfeels.Time`. None of my dependencies know what that is. Then I have to rewrite all my code, and dependencies, to use this weird, non-standard date/time object. This would be a great place for an interface instead of a concrete type, but that ship has sailed.

Instead of rewriting every package that uses `time.Time`, I could rewrite my entire parser to not use any private members. It would just use the public constructor for `time.Time`. It turns out this is pretty complicated and time consuming, and it runs the risk of introducing new bugs with very little upside. 

I could fork the entire Go stack with my new standard library and force everyone in engineering to only use _my version of Go_. That seems ... unreasonable.  

In the end the solution I chose is simple, and kind of hacky:

```
// Convert our package-local privateTime struct to the stdlib Time struct
func privatetimeToGotime(t privateTime) time.Time {
  return *(*time.Time)(unsafe.Pointer(&t))
}
```

I made a copy of the `time` package with my changes to the parser, and a shim to convert between my package's `privateTime` type, and the standard library `time.Time` type. This works because `time.Time` has _exactly_ the same definition as `privateTime`. We can mess with the private members of `privateTime` in our forked standard library parser, then convert it back to a normal `time.Time` with this method. 

Depending on this kind of coupling is normally _not good:_ it creates spooky action at a distance. Changing one implementation affects the other in a way that isn't apparent until runtime. Fortunately, we have unit tests and CI for this. When the definition of `time.Time` changed between Go 1.3 and Go 1.5, we noticed immediately and fixed the definition. We control the version of the standard library our code builds against, and we can look at the source for changes. 

The entire "hack" is encapsulated in a single package, and has no impact on our dependencies and the rest of our code base. You can see that our forked implementation of `time.Time` isn't even publicly visible - it's `time.time`. The public package API exclusively uses the `Time` struct provided by the standard library, to avoid confusion in the calling code.

In the end what felt like a gross hack (and may still seem like a gross hack to some people) ended up being a reasonable trade-off to make customers happy and keep engineering moving forward. With a small team (just two people working on the ingest side!), making the right trade-offs is crucial for success.

P.S. Please just fix this in the standard library so this whole blog post is pointless.
