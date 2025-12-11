+++
date = '2022-05-27'
draft = false
title = '.NET is not LTS'
+++

New languages and frameworks all seem to have one thing in common: the rate at which they try (or have) to innovate. Drop a framework, look away for a moment and you come back to a rewritten stack of entirely new concepts. The space between version one and version two is almost unpalatable.

Modern software development almost demands a rapid fire of changes. There's no room for breathing when everyone promotes "move fast and break things". It's a recursive pattern fueled by us and us alone. Developers demand this and yet, at the same time, developers demand stability, performance and convenience.

Microsoft adopted such a "most fast and break things" policy when introducing .NET Core. Way back when .NET Core 1.0 was revealed to a much smaller parade of people than today's releases, little did I know that the entire landscape of .NET development was about to change. Up until this point it was comfy - you'd grab a cushion and deal with the quirks and bloated nature of .NET Framework, which only ran on Windows, but you'd be safe in the knowledge that, whatever you'd write would work on a target machine. .NET Framework would be preinstalled and, whatever you end up deploying, would work in 5 or 10 years time. I still have applications running .NET Framework 4.0 in production today, almost 13 years after the fact.

Today; we're faced with a put up or shut up attitude. .NET (Core) is moving at a pace which demands C# versions update yearly and new or breaking API changes (because it's a major version, duh) are made regularly. Look away for just one year and your upgrade path is littered with nuances of the version you might have skipped. While Microsoft's documentation is likely some of the best programming documentation out there, most of it is lost to the void in unindexable YouTube videos. Good luck finding that. Not only does the pace welcome breaking API changes, but even Microsoft themselves seem to lack the ability to keep up. MAUI was slated for release alongside .NET 6 in November of 2021. A little over six months later, MAUI finally hit what the developers considered a "ready" product. We won't talk about MAUI in this post, but there's plenty to cover over how MAUI in itself is a fundamentally flawed and developer-hostile framework.

Small updates to .NET (Core) now break older frameworks. WPF likely suffers the most here. Tooltips randomly disappear. DPI scaling breaks.

Of course these issues happened before Core and they'll happen long after the next big Microsoft framework is made; yet this has become so much more prevalent and pronounced as .NET becomes more popular, more demanding and the policy remains the same: move fast and break things.

.NET's yearly release cadence is taxing on most, from seasoned C# developers to novices hoping to make the next big thing with .NET as their chosen weapon. Enterprises hoping to make use of .NET will likely opt for the long term support, or LTS, releases instead. These are *bigger* releases of .NET that will be supported longer than just one year. Three years at most.

Three years is a drop in the ocean for a bigger corporate project. Of course .NET won't just stop working at the point it's no longer officially supported by Microsoft, but bullish blog posts, even emails from Azure, push developers to upgrade their projects. This leads those same developers down a rabbit hole of not only finding the required time (and often funding) for said upgrade, but now documentation that previously implicitly relied on a developer upgrading their .NET application year-on-year is sat needing to be stitched together in the right order so that we can bump from one major version to another.

These LTS releases are an attempt at trying to instill confidence in an ecosystem that has long dropped prolonged stability and usability in favour of moving fast and breaking things.