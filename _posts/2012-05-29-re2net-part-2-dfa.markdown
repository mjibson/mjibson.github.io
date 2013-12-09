---
layout: post
title: "re2net - part 2: DFA"
date: 2012-05-29 15:54
comments: true
categories: re2net
---

I have [added](https://github.com/mjibson/re2net/commit/4f5f274fb6299c703d70541af795918371f0bdd6) a DFA state machine to the re2net library, as described [here](http://swtch.com/~rsc/regexp/regexp1.html). This method computes the DFA states on demand, which makes subsequent matches with the same instance faster. The crude benchmark below shows run times for the NFA (as in [part 1](http://blog.mattjibson.com/2012/05/re2net---C-RE2-implementation-part-1)), DFA's first run, DFA's second run, and C#'s Regex library as a comparison.

The results below show that the DFA is about an order of magnitude slower than the NFA on the first run (as expected, since the cache is being computed), but an order of magnitude faster on the second run (since the cache is being used).

Until now, this has been an academic exercise to learn C# and build a simple regex parser. With that done, the next step is to support the full regex syntax. The [RE2](http://code.google.com/p/re2/) library is this project, but I'm going to use [Go's regexp package](http://code.google.com/p/go/source/browse/#hg%2Fsrc%2Fpkg%2Fregexp) since I think the code will be easier to read, and it's the same implementation. This is all assuming I maintain interest.

Column headers, all times in seconds.

n, NFA, DFA, DFA2, C# Regex:

<pre>
01: 0.0046062, 0.0025143, 0.0000043, 00.0000278
02: 0.0000148, 0.0000108, 0.0000024, 00.0000040
03: 0.0000222, 0.0000231, 0.0000034, 00.0000034
04: 0.0000145, 0.0000216, 0.0000077, 00.0000021
05: 0.0000170, 0.0000355, 0.0000049, 00.0000030
06: 0.0000213, 0.0000476, 0.0000102, 00.0000052
07: 0.0000334, 0.0000742, 0.0000064, 00.0000092
08: 0.0000395, 0.0000971, 0.0000077, 00.0000176
09: 0.0000504, 0.0001212, 0.0000080, 00.0000346
10: 0.0000621, 0.0001552, 0.0000086, 00.0000702
11: 0.0000766, 0.0001821, 0.0000086, 00.0001184
12: 0.0000899, 0.0008207, 0.0000092, 00.0002622
13: 0.0001011, 0.0002600, 0.0000092, 00.0004787
14: 0.0001178, 0.0003105, 0.0000111, 00.0010533
15: 0.0001252, 0.0003729, 0.0000120, 00.0020185
16: 0.0001759, 0.0004468, 0.0000160, 00.0041630
17: 0.0001945, 0.0004914, 0.0000120, 00.0088549
18: 0.0002310, 0.0005789, 0.0000157, 00.0160122
19: 0.0002409, 0.0007215, 0.0000139, 00.0318414
20: 0.0003145, 0.0008251, 0.0000170, 00.0634503
21: 0.0003732, 0.0009216, 0.0000148, 00.1416363
22: 0.0004308, 0.0009741, 0.0000185, 00.2809800
23: 0.0004156, 0.0010700, 0.0000160, 00.5715502
24: 0.0004849, 0.0011817, 0.0000204, 01.1391405
25: 0.0005489, 0.0017962, 0.0000216, 02.2826699
26: 0.0005520, 0.0014535, 0.0000228, 04.5484486
27: 0.0010332, 0.0016941, 0.0000213, 09.0551722
28: 0.0010230, 0.0018286, 0.0000216, 18.4857312
29: 0.0010944, 0.0019780, 0.0000207, 37.1557184
</pre>
