---
layout: post
title:  "EvtSubscribe and Event Forwarding"
date:   2016-08-07 00:00:00
categories: golang windows eventlog event forwarding evtsubscribe
---

I recently got some feedback from someone who was trying to use [gowinlog](https://github.com/scalingdata/gowinlog) to subscribe to the "Forwarded Events" channel on a Windows server. It was not going as planned.

For context, "Forwarded Events" is a special channel - the default channel where events from [Event Subscription](https://technet.microsoft.com/en-us/library/4aa6403f-d4b8-43a4-a70d-ceb7f88c524e) are stored. These events are pulled (or pushed) from remote machines and stored on a single system to simplify administration. "Forwarded Events" shows up in the Event Viewer just like your standard, local logs.

The mention of logs from a remote machine made me wonder if this was going to be possible without some code changes. gowinlog is mostly a thin wrapper around [EvtSubscribe](https://msdn.microsoft.com/en-us/library/windows/desktop/aa385487(v=vs.85).aspx) and related APIs. One of the arguments to `EvtSubscribe` is `Session`, a connection to a remote system which should allow you to subscribe to logs without configuring forwarding. Or that's how I read it. I haven't plumbed the depths of remote Windows logging (I've barely skimmed the surface of local Windows logging), so this was new territory.

Running `Get-EventLog`from Powershell didn't show a "Forwarded Events" channel. That's bad news. I read the man page for `Get-EventLog` and realized it's deprecated and I should be using the intuitively-named [Get-WinEvent](https://technet.microsoft.com/en-us/library/hh849682.aspx) so I can see fancy Windows Event Log logs as well as "classic" logs. Cool story. `Get-WinEvent` showed the same list of channels as the Event Viewer, including "Forwarded Events". Dead end.

Finally, I came across a [Splunk post about Event Forwarding](http://blogs.splunk.com/2014/02/03/forwarding-windows-event-logs-to-another-host/), which mentions offhandedly that the default forwarding log is "ForwardedEvents" (all one word). In a fit of desperation, I try subscribing to "ForwardedEvents" and everything fits together. Hooray.

Takeaways:

- You can't have spaces in your channel names when you call `EvtSubscribe`. Just take the spaces out and everything will work. This isn't in the docs as far as I could see, and it's not reflected in the Event Viewer or `Get-WinEvent`.
- `EvtSubscribe` *does* work with Event Forwarding out of the box. There might be more corner cases I'll cover later.
- You should always use `Get-WinEvent` and not `Get-EventLog` to work with event logs.
