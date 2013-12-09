---
layout: post
title: "Courage and Knowledge"
date: 2013-01-03 00:58
comments: true
categories: development
published: false
---

One of the most frequent impediments to action in ignorance of the results. This is not ignorance that there will be results. It is knowledge that there will be, but not **what** they will be. This occurs often enough to me as a software developer that I thought others might find it interesting.

## Breaking Possibilities

Here is what happened to me today (all true): I was given a custom support request and asked to handle it. It involves a part of the code I've never looked at. It is a large, complicated, money-handling system that would take me longer than I have to satisfactorially understand it. I found a fix. It worked on my machine. The person who understands and wrote the code is out on paternity leave and won't be back for a while. None of the other developers in our dev chat room responded when I asked about this issue.

At this point I had two choices:

1. Be conservative (I will avoid the word coward) and wait until someone else OKs the change (which could potentially take days or weeks--too long).
1. Be brave and just do it. The "it" in this case was a manual, hand-typed SQL update on our production database. Brave is the correct word.

Development process at my company is fairly lax:

{% blockquote @Nick_Craver https://twitter.com/Nick_Craver/status/278223940562874369 %}
We keep hearing the term "unit test" from @jarrod_dixon in our hangout, anyone know what the hell he's on about?
{% endblockquote %}

{% blockquote @marcgravell https://twitter.com/marcgravell/status/278224349243248641 %}
I assume it is using the word "unit" meaning "1". We have one test: "is it awesome?"
{% endblockquote %}

We are entrusted to make decisions. I generally fall in the break things camp, so I went ahead with my change. I haven't seen anything break yet _phew_. But I was still **stalled by fear** of what might happen.

## Partial Knowledge

The problem is a lack of knowledge. If we understood the system, we would have little fear since we could forsee most issues. Obviously, we are expected to learn whichever system we are working on. This could be difficult for various reasons. Here are some and proposed solutions.

1. The person who wrote it no longer works there (or does and just forgot) and left little documentation about design decisions. The solution here is that **you** must become the new owner. Breaking changes will happen, but that is acceptable (since no one else, by definition, could do any better).
1. It involves other systems about which I know little. SQL intricacies, Javascript events, CSS rules, C# dynamics, for example. This one is harder. The solution here is something like "read the book about it", which can take days or weeks that you do not have.
