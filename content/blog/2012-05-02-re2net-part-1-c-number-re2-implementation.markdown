---
title: "re2net - part 1: C# RE2 implementation"
date: 2012-05-02
slug: "re2net"
---

After reading Russ Cox's [regular expression articles](http://swtch.com/~rsc/regexp/regexp1.html), I became interested in porting his [RE2](http://code.google.com/p/re2/) library to C#. These posts will describe my effort to port the entire library, which I will do in steps. I know that one could just link to the RE2 library, but I am porting this as an academic exercise to learn C# and regular expression parsing.

The full RE2 implementation is somewhat large. So, for now, I have started with something simpler: porting [nfa.c](http://swtch.com/~rsc/regexp/nfa.c.txt), which is now [committed](https://github.com/mjibson/re2net/commit/4deade8190159843ee512e8b99da5fbaa68fa1e4).

### Performance

There is a simple script that generates the a?^(n)a^(n) regexes and compares the performance between this simple nfa implementation and C#'s Regex class. The results (posted below) conform exactly to those posted in the article.

n, nfa match time (s), C# Regex match time (s):
<pre>
01: 00.0040588, 00.0000321
02: 00.0000299, 00.0000027
03: 00.0000188, 00.0000018
04: 00.0000142, 00.0000021
05: 00.0000191, 00.0000030
06: 00.0000244, 00.0000052
07: 00.0000343, 00.0000092
08: 00.0000405, 00.0000231
09: 00.0000569, 00.0000343
10: 00.0000671, 00.0000683
11: 00.0000794, 00.0001336
12: 00.0001039, 00.0002665
13: 00.0001218, 00.0005337
14: 00.0001447, 00.0010694
15: 00.0001673, 00.0021296
16: 00.0005155, 00.0047231
17: 00.0002180, 00.0085827
18: 00.0002226, 00.0167507
19: 00.0002501, 00.0322935
20: 00.0004020, 00.0742515
21: 00.0005820, 00.1423361
22: 00.0004060, 00.2887820
23: 00.0004518, 00.5772489
24: 00.0005170, 01.1505112
25: 00.0005616, 02.3019515
26: 00.0006077, 04.6177123
27: 00.0009324, 09.1860805
28: 00.0007561, 18.5854136
29: 00.0010960, 37.6036043
</pre>
