---
layout: post
title: "Bosun: Technology Review"
date: 2014-11-13 00:00
categories: [bosun, angularjs, typescript, go]
---

[Bosun](http://bosun.org) is a monitoring and alerting system I have been
working on at Stack Exchange. We started work on it about a year ago. I would
like to discuss the technology choices made then, how they have fared, and which
I will continue to use or stop using. Let's go front-to-back.

# CSS

No pre-processor, just [Bootstrap](http://getbootstrap.com/) and hand-written
CSS. I've been using this method for years and am still ok with it. Probably
because I don't write that much CSS to make using less valuable. I also don't
minify. No exciting news here. I'll keep doing this unless I start on a larger
project with more serious design.

# AngularJS

I've been using [AngularJS](https://angularjs.org/) for recent projects and
have, in general, liked it. Bosun was the first time I had strong negative
experiences with it. For small projects, I find AngularJS understandable. There
aren't too many controllers or directives to confuse life. Bosun grew into a
size that made AngularJS confusing, though. It had too many directives and
controllers and services to know what was going on. I didn't know which things
were in which scope. The batarang is there, yes, but because of Angular's scope
inheritance stuff, it was really difficult to see where things were coming from.
Hours were lost trying to figure out where the data I wanted was, and how to get
it where I wanted it to go. All of this is due to Angular's unique decision to
embed execution instructions in HTML, which means you can't put breakpoints on
them to inspect and see what's going on. Yes, there are ways around these
things, but it was too opaque to be fun anymore. Performance is also becoming an
issue, and I'm spending more time than I want trying to get Angular to draw less
things and update less times so I can render a large page. I'm not going to use
AngularJS again for a new project. My current interest is
[React](http://facebook.github.io/react/), and it appears they solve a lot of
these problems, though I assume I'll find similarly annoying things about it.

# JavaScript

The project I worked on previous to Bosun was an internal app for our sales
team. I tried out [CoffeeScript](http://coffeescript.org/) for the first time
and liked a lot of it. But I found that I outsmarted myself and had a much
harder time than usual reading my own code. Further, it generated things
different than my intention enough times that it was almost dangerous. I wanted
to try something else.

For Bosun, I went with [TypeScript](http://www.typescriptlang.org/). Having
switched from Python to Go in the last few years, I was back on the type-check
train and wanted that for my JavaScript. TypeScript does indeed do an excellent
job at checking types. What it is poor at, though, is helping the user to know
what types to type (perhaps the Visual Studio experience is better, but I was
using a Mac). Many times I didn't know what type a thing was or should be, so I
just ended up using 'any' everywhere, removing much of the benefit. And too many
external libraries (although they all have user-submitted definition files) were
missing small things. It didn't help more than it hurt, and came in about a
wash. Overall experience was not better than straight JavaScript, so I don't
plan on using TypeScript for new projects. If it had something like godoc, which
lists in a web browser all available functions, classes, and their types, then I
may consider it again.

I'm currently in the market for a good JavaScript+framework solution. I'd prefer
to stay out of the node-based JavaScript processors. I tried to make bindings
for React to GopherJS (a Go -> JavaScript transpiler), but [had
showstoppers](https://github.com/gopherjs/gopherjs/issues/118). My next try will
be JSX with React, which I may end up liking.

# Go

The backend is all [Go](http://golang.org/). I love Go the more I use it; it was
a great choice for Bosun and scollector (a small metric-collection agent that
runs on all our servers and pushes data to Bosun). The features that made Go
pleasant to work with are the same that are being lauded in many current talks
and posts. The concurrency made processing lots of data from many places easy.
Its simpleness and lack of magic made understanding exactly what was going on
possible. Its cross-platform nature made development and deployment quick. Its
standard library provided much functionality and starting points for new
packages (for example, we used the text/template/parse package as a starting
point for a parser for a new grammar).

The pain points for Go were significant, but overcomable. Since we are a Windows
shop, we needed lots of data about Windows machines. This means using WMI to
query for data. WMI means COM calls, which are not fun in any language. But C,
C++, and C# all have nice wrappers and example code. Go has no such code. As a
result, we had to [write our own library](https://github.com/stackexchange/wmi)
which was plagued with memory corruption and memory leak bugs for a long time.
Those bugs are now fixed, but the library is still limited and doesn't support
everything we would like it to. Go's newness and Unix-centeredness has hurt
here.

The other Go problem we frequently ran into was dealing with concurrency and
locks. Since we have lots of data coming in, and a few other processes querying
that data, we had to employ locks to prevent more than one go routine from
accessing and mutating maps and other data structures. We ended up
double-locking (and thus deadlocking) frequently. These problems could probably
have been prevented by a more diligent design, but I designed them mostly myself
and without code review. But the point is that, although Go makes concurrency
easier, it does not prevent problems outright.

# Database

The other huge benefit to Go is that it is powerful and low enough to not need a
separate system to store its data. Also, for a system that needs to alert when
things go down, Bosun could not depend on Redis or SQL. Persisting internal
state to disk with [gobs](http://golang.org/pkg/encoding/gob/) (Go's native
serialization format) was quick and easy. It's not scalabale, but it's a decent
start. There are multiple packages that will provide the database features we
need and be in pure Go (I'm thinking of [ql](https://github.com/cznic/ql),
[kv](https://github.com/cznic/kv), and
[goleveldb](https://github.com/syndtr/goleveldb)). That these projects exist
without external dependencies is one of my favorite things about the community
that Go has inspired.

# Conclusion

I enjoy trying new technologies. The best part is they sharpen my thinking about
how I like to work, and hopefully add some experience and wisdom about larger,
general programming problems.
