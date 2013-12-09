---
layout: post
title: "Fast Fourier Transform in Go"
date: 2011-12-09 15:59
comments: true
categories: [go, fft, go-dsp]
---

I just completed all the basic functionality for a fully-working [FFT](http://en.wikipedia.org/wiki/Fast_Fourier_transform) implementation. It supports inputs as arrays of real or complex values, with the inverse transform, too. It is part of my [go-dsp project on github](https://github.com/mjibson/go-dsp).

Install with:

`$ goinstall "github.com/mjibson/go-dsp/fft"`

Some example use code:

    package main

    import "github.com/mjibson/go-dsp/fft"
    import "fmt"

    func main() {
            fmt.Println(fft.FFT_real([]float64 {1, 2, 3}))
    }

### Brief introduction to the fast Fourier transform

Input arrays of length a power of 2 use the radix-2 FFT algorithm (the butterfly one). Inputs of other sizes (non power of 2 and prime lengths) use the Bluestein algorithm. The Bluestein algorithm is interesting because (in contradistinction to the other FFT algorithms for non power of 2 lengths) it works on prime and non-prime lengths. Other solutions require one each for prime and non-prime lengths, so you end up with 3 total algorithms. The purpose of this project was just to get something working, and worry about performance and optimization later.

So, back to Bluestein's algorithm. It works by starting with the definition of the discrete Fourier transform (the fast Fourier transform (FFT) is a way to compute the discrete Fourier transform (DFT)), and then doing some algebra on it. This results in an equation that does an operation known as convolution. Convolution is really hard. So, someone created a way to change convolution into multiplication (really easy). That way is...the Fourier transform! If you take the Fourier transform of two arrays, then multiply the result, and then take the inverse Fourier transform, you have just done convolution. So, you can do the convolution you got from messing with the DFT equation by adding some zeros to the end of the array so it's of length a power of 2, and then use the radix-2 FFT algorithm to do the FFTs, multiply, then inverse FFT back, and you have the DFT of an array of any length.
