---
title: "Perceptual Diffs at Stack Overflow"
date: 2013-06-11
slug: "pdiffs"
---

# pdiffs are worth the time

After hearing about Brett Slatkin's discussion of [perceptual diffs](http://www.youtube.com/watch?v=UMnZiTL0tUc), I wanted to try it out on [Stack Overflow Careers](http://careers.stackoverflow.com/). I wasn't convinced it would be worth the time to implement. But then I started keeping a list of the bugs it would have found. These bugs were hard to find and solve because they were so strange. A CSS refactor missed some rare pages. Changing newlines in our files from `\r\n` to `\n` broke multiline strings on localized pages (checksum changed, so it couldn't find the localized strings). Other obscure things. Each time this happened I thought that pdiff would have found it before we did. Eventually it became clear that it was worth the time investment.

# implementation

I decided to use the same deployment style as perscribed in the videos: when new code is pushed, UI tests run, and pdiffs run separately but in parallel. This wouldn't add any time to our commit-build-test-deploy cycle. And we could see if there were any problems before deploying to production.

<img style="width: 100%" src="/images/pdiff-build.png">

Our pdiff step is implemented as its own nunit test class in its own category (so it can be run separately from our UI tests). We have a list of URLS and run a selenium process to screenshot each of them in all of our supported languages. Some pages need user input, so we support running arbitrary code before screenshotting. Screenshots are saved on a network drive addressed by their URL, some other identifying string, and build number.

The second step performs the actual diffing. A process recurses through the directories and compares any files with identical URLs and identifying strings, sorted by build number. We pdiff the latest two. There is a free implementation of pdiff online, but that requires calling out to an executable. It turns out writing a naive pdiff implementation is about [10 lines of code](https://github.com/mjibson/pdiff/blob/a5974e7e175f2ba53987c53b1596cb27fc85e5a6/pdiff.go#L67), so we just implmented it ourselves. It's trivial to read in two images and compare each pixel. To generate the diff image, start with black and write red if the pixels aren't the same. Now we have a list of pdiffs. A static html file showing the pdiffed images is generated. If there are differing images, we have a chat bot (named pdiffy) that announces to our developer room for review.

<img src="/images/pdiff-chat.png">

# results

It was just a few days of work to get this all done. The week after it was running, it found an unpleasant and invisible bug. We were releasing a new feature that needed to turn on at a certain date, and we were off-by-one on the year. pdiff caught it before it could go out.

There are problems. First, it generates lots of false positives. This is because some scripts take a few seconds to initialize, so we have to wait a while before taking a screenshot. Sometimes it's not enough, so subtle things change. I'm bumping up the wait times until these stop. Second, since there's no real server, just some static html pages, it's not super friendly to use. Ideally, a pdiff server would maintain a list of images that needed review instead of just telling us something changed. Then we wouldn't duplicate work of reviewing images some one else already has. This requires a full server that knows what's going on. Haven't had time for this yet. Finally, we are currently only running pdiff on logged-out, static pages. Pages with dynamic content (i.e., showing random jobs) are not being tested since we have to add server support so it always renders with the same content and user state.

For the future, I am planning on making a REST-like pdiff server running on app engine. I have a [project started already](https://github.com/mjibson/go-pdiff) that is partially functional. I chose not to use [dpxdt](https://github.com/bslatkin/dpxdt) because I like running things on app engine. And now that [GCE](https://cloud.google.com/products/compute-engine) is generally available, it's possible to use that to take screen shots. And the go-pdiff backend is just an interface, which means it's possible to add a redis or sql backend, and make a vagrant config so it's easy to spin up your own internal pdiff server if you don't want to use the app engine hosted one.

# conclusion

Perceptual diffs are not that hard to implement. There are not lots of drop in solutions now. But it is so straightforward to implement, and can catch so many hard kinds of bugs that nothing else can, that it is worth the time and effort.
