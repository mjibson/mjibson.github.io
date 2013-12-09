---
layout: post
title: "Careers Localization, part 1: Why Roll Our Own?"
date: 2013-02-27 15:52
comments: true
categories: [careers, localization]
published: true
---

Localization is a difficult feature to add to a website. A few months ago, we at [Stack Overflow Careers](http://careers.stackoverflow.com/) localized into [German](http://careers.stackoverflow.com/de/). This took months of effort from much of the team. I would like to describe our design and implementation over a few blog posts.

This post will discuss our design choices.

## Requirements

Localization consists of

1. indicating text that should be localized
1. translating it
1. showing the correct translation to the user

There are various existing solutions in most programming languages that can do this. We looked at some, but none were able to meet all of our design and support requirements:

1. C#
1. JavaScript
1. Razor views
1. data attributes for localized error messages on form validation
1. markdown formatting (so that no HTML is ever sent to translators)
1. dynamic text replacement
1. pluralization-aware
1. gettext-style (English text in a function - no resource files and identifiers)

We discussed some of these requirements with others who had previously localized their sites. Some had decided to forfeit features due to difficulty of implementation (pluralization, dynamic text). Supporting each of these requirements is difficult, so we understood the decision to omit them. However, we were not willing to do the same.

## Our Solution

The other solutions missed some our of requirements, or functioned in a way we didn't like. We were, however, heavily influenced by existing work, like that of [Daniel Crenna](https://github.com/danielcrenna/i18n). We have a long history of rolling our own software when the 90% solution isn't enough, and that tradition was continued here. We ended up writing the entire implementation from scratch. This includes designing an API, extraction process (to determine what strings the translators should translate), and localization engine to support numbers, dates, and translated strings. This allowed us to have a unified pipeline for text extraction and processing (i.e., the C# and JavaScript implementations could be identical at each step of the process), and give us the power and speed we wanted in the implementation.

As will be described in future posts, our implementation met all of our requirements, but with some limitations. Localization is hard, and doing everything is near impossible. If we wanted something better than the 90% solution, then what we have now is maybe in the 95% range: there are still missing features that we haven't figured out yet.

[Part 2: The API](/blog/2013/02/28/careers-localization-part-2-api/)
