---
layout: post
title:  "Lessons Learned in a Distributed Organization"
date:   2017-01-22 00:00:00
categories: teams distributed work-from-home remote-work remote
---

I want to change things up a little for the first post of 2017, and reflect on the past two years.
I've been working at Rocana, where we're building an entirely distributed company. 
There's no head office, there's no satellite offices - every person works from home 
(or wherever they're comfortable) distributed across four time zones.
We see each other face-to-face every 3-4 months when we fly in for team meetings, 
but for the most part collaboration is over Slack, video chats and Github. 

Working remotely has a lot of benefits. I love being able to live in Ottawa, 
while working with and learning from brilliant people across the continent.
Being able to travel and work from other places is a great perk,
although I don't think I'm cut out to be a "digital nomad" (ugh, millenials).
The flexible schedule and the absence of a commute make it easier to make 
the best use of your time. In Ottawa I'd much rather do my early-spring
marathon training at noon than 7PM, thanks.

A remote organization does have to do things differently. There are some things
that are simply more challenging when everybody isn't working in an office together. 
Here are some of the things I've noticed and the ways we've been able to work around them: 

**Have a single source of truth** - Working remotely you rely heavily on your tools
to keep everyone on the same page. This means JIRA, Slack, Github, Google Docs, a wiki.
You need to be able to keep track of the decisions you've made as a group, and you
also need to be able to find those decisions. If every meeting results in a new set of
minutes, new people on the project need to read all of those minutes to catch up.
If you make decisions in Slack, people have to read all of the Slack history to keep up.
If you give a presentation outlining a project, those slides are stale as soon as you share them.

Maintaining a living design document for a given feature or section of the product can 
provide an up-to-date survey. This gives people outside the team the ability to check 
in quickly, it helps onboard new team members and keeps a record of the decisions that 
have been made. It's also a great place to keep acceptance criteria and other high-level 
goals that the team will need to refer to throughout feature development.

**Sync up frequently** - When you're working from home it's easy to focus on a specific problem
and really dive in. You get in the zone, and when you run into roadblocks you
narrow your focus and try to work past them by yourself. This is my default style of working: take a big 
chunk of work and churn away for a week or two with my head down. This is obviously
an anti-pattern in most development shops, but when working remote it's 
an especially difficult habit to break. The end result is giant, risky, unreviewable
pull requests - they're hard to read, and it's a big setback if you can't merge them.

A way to work around this is to communicate more often: brainstorm up front
with the team before kicking off the work. Share unfinished code and discuss
the hurdles that come up during development. Be explicit about the known unknowns,
and expect that you'll run into unknown unknowns. Make small, incomplete prototypes
to illustrate how you're approaching a problem. It takes a lot of vulnerability to work
in this style - showing your work before you've solved all the problems - but it's 
also a great chance to learn and benefit from your coworkers' knowledge.

**Disagree face-to-face** - Text is a terrible way to communicate when you disagree with someone. 
There's no emotion, there's no nuance - the best we have are emojis. Emojis are not a good way
to tell someone you think they're wrong. I've seen PRs with 50+ comments on them,
which seems like a clear sign that communication has broken down (or the PR is way too big).
Even for design documents, a back-and-forth of asynchronous comments can get bogged down in
detail, where a face-to-face conversation could lead to both parties taking a step back.

Especially for larger, more subjective disagreements it's worth talking to someone one-on-one.
Ideally this would involve video, but at least a voice chat. Both sides can lay out their positions,
and hopefully arrive at a decision. These decisions should be recorded for the sake of 
transparency, but the entire sausage-making process doesn't have to be open to the public. 

**Take time off** - The flexibility of working remotely means you can do it from anywhere!
But you shouldn't. I've found myself checking Slack when I'm out at dinner with friends,
when I'm on a chair lift at a ski hill, half-way through a three-hour run, and many more
inappropriate times. It takes a lot of discipline to disconnect, both at the end of the day
and during vacation. 

One solution to this is personal: I turn off notifications when I'm done working. I
remove the entire Slack app from my phone when I'm on vacation. It's a tough habit to break
when you're a workaholic who _wants_ to always solve everybody's problems.
The other solution is organizational: have clear expectations. If someone needs to deal
with customer escalations on a Saturday night, that person is on-call. They should know
when they're on-call, and they should know when they get to stop being on-call. Even if 
it's just reacting to other teams within the organization, it's important to be explicit about
when people should stop thinking about work.

Working effectively as a remote team is something we're constantly working to do better at.
If you have advice and best practices for building distributed teams I'd love to hear from you! 
