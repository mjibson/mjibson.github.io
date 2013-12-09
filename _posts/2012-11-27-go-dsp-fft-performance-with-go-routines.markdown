---
layout: post
title: "go-dsp FFT performance with go routines"
date: 2013-01-04 03:30
comments: true
categories: [fft, go, go-dsp, performance]
---

[go-dsp](https://github.com/mjibson/go-dsp) has been around for almost a year now. Recently I have been working on performance improvements. The low-hanging fruit was easy (removing duplicate calculations, smarter array copying, efficient data reordering with bit reversal). The difficult part was getting go routines to work as intended. That is, to have them improve the performance of go-dsp. This turned out to be more difficult than I expected.

## Why the fast Fourier Transform is parallelizable

[{% img right /assets/images/DIT-FFT-butterfly.png 300 %}](http://en.wikipedia.org/wiki/File:DIT-FFT-butterfly.png)

The FFT is paralellizable because of how the "fast" part is implemented. Given an input of length `L`, if there exist integers `M` and `N` such that `L = M * N`, then the original problem (one transform of size `L`) can be restated as `M` problems of size `N`. These `M` problems can be run in parallel. For example, say I have an input of size 8. I can reform this as 2 inputs of size 4. These 2 inputs can be run in parallel.

## Attempts

Go's easy support of go routines was the obvious solution here. I went through [a few solutions](https://github.com/mjibson/go-dsp/tree/mp-test) until I found one that was **not** slower. What I discovered was that, although the FFT is highly parallelizable, setting up parallelization can easily take more time than it saves. The actual unit of work that is done is just multiplying two pairs of numbers and saving the result.

My first idea was to use wait groups. This involves a synchronized counter. One go routine is spun up per block (there are `M` blocks, as described above) and the counter incremented. The go routine decrements the counter when it is done. We wait until the counter is back to zero. Since the number of blocks varies from `2` to `L / 2`, this means that for about half of the time, so many go routines are spun up to do a very tiny amount of work that overall runtime increases. Ok, so, only do the wait group solution if it'll actually be faster. I ran some tests and found out that (on my machine), if the block size is over 128, it's worthwhile to spin up a new go routine. Remember that this solution was always using a single go routine when we were on smaller block sizes, ignoring any potential multicore speedup.

The second idea was to use worker pools. Since the main problem is the creation (not use) of go routines, spin up as many as we will need up front and then send work off to them. From testing I found that putting the number of workers at the value of [`GOMAXPROCS`](http://golang.org/pkg/runtime/#GOMAXPROCS) worked out well. Each worker's job is to multiply the number pair described above. So, for each of the `n log n` iterations, we are sending off `L / 2` jobs. This ended up performing almost exactly as good as the above solution for larger block sizes, but had the added benefit of working with smaller block sizes, too. I guessed the reason for the lack of more speedup was the communication overhead. The jobs were sent using channels, which aren't free.

The final solution addressed that problem by changing the number of jobs from `L / 2` to the block size (which, as a reminder, goes from `2` to `L / 2`). So only during one iteration are we sending the same number of jobs over. Almost always are we sending less. Previously, the jobs were specified by an index. This new solution instead specified a min and max index. The subsequent indicies are calculated in the worker itself. This results in much less channel overhead and distributes some work (index calculation) out to workers.

## Results

The original, single-thread solution contains no multicore logic. It benchmarked at around 505ms. The graphs below show performance at `GOMAXPROCS = 6` and a FFT size of `2 ^ 20 = 1048576`.

{% img /assets/images/fftmp-1.png %}

Above are results for the first two attempts. The blue line is the original, single-thread control. The green line shows the very poor performance at small block sizes. The red line shows similar performance as green for large block sizes but that it was able to handle smaller block sizes better (but still not great). Minimum runtime was 1.7x faster (267ms minimum).

{% img /assets/images/fftmp-2.png %}

Here we see the final solution with indexed worker groups. Minimum runtime was 252ms (2.0x speedup). Not the 6x increase I wanted, but it's not bad.

The final code (using the indexed worker group solution) is now [available](https://github.com/mjibson/go-dsp).
