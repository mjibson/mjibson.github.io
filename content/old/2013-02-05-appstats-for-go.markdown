---
title: "Appstats for Go"
date: 2013-02-05
slug: "appstats"
---

I would like to announce the release of appstats for go. Installation instructions are [available on godoc.org](http://godoc.org/github.com/mjibson/appstats). I'd like to thank Dave Symonds for [his help](https://groups.google.com/forum/?fromgroups=#!topic/google-appengine-go/laOsfI1dSGY) on this project.

A [demo site](http://schalmei-go.appspot.com/) is available.

[<img style="width: 100%" src="/images/appstats-timeline.png">](/images/appstats-timeline.png)

[Appstats](https://developers.google.com/appengine/docs/python/tools/appstats) is an incredibly useful library for the python (and java) runtimes. The go runtime has had no similar library, adding to the difficulty of developing significant apps. I'd like to describe a bit about how appstats is implemented (applies to python, as well) and where I think the go runtime is today.

## implementation

Appstats for go was implemented by copying the python HTML templates, examining the source, and attempting to make something work in go.

### intercepting the data

In order for appstats to work pleasantly, it needs to automatically populate data from the HTTP request and intercept all RPC calls to app engine's services (datastore, memcache, etc.). This is done by a wrapper that provides an `appengine.Context` instance to a handler function. Both the `http.ResponseWriter` and `appengine.Context` variables are actually appstats structs that forward importart calls and record timings and data. Go routines are fully supported: all of the internals are thread-safe, and appstats waits for all RPC calls to complete before serializing and saving its data.

### persist to memcache

I had always wondered how python appstats stored its data. It was obviously in memcache, but it was interesting how it was able to so quickly store so many requests and fetch them all. How did that work? The timestamp on a request (well, just `time.Now()` when appstats starts) is used. The current count of microseconds is the memcache key to a request. If it overwrites another request, that's ok. For example, a request that happened at `04:10:40.40567` gets the key `40500`. The final two digits are converted to 0 so that we can limit the number of keys to 1000. Then, to fetch them all, appstats generates all 1000 possible keys and requests them from memcache. Only existing keys are returned. Each request stores a partial (for the index dashboard) and full (for the details page) item in memcache, with the type ("part" or "full") appended to the key name.

### things it doesn't have

There are a few things that python appstats does that this implementation does not (yet). Some may not be possible ever, and some may require Google to help us out a bit:

- full RPC cost information (only writes cost, currently; reads and small ops not yet implemented)
- stack traces with variables and values at each stack frame
- protobuf examination for better details (in the "Response" and "Request" lines of each RPC call on the details page)

## today's app engine go runtime

The two killer features of the python runtime on app engine are NDB and appstats. I have been refusing to make a serious app with the go runtime because of this lack. To address those concerns, I have created [goon](https://github.com/mjibson/goon) (an NDB-like library for go) and appstats. With the addition of these two libraries, I believe **the go runtime can now compete with python**.
