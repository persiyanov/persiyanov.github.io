---
layout: post
title: "Weeks 3-8 in GSoC, Gensim project"
---


# Greetings!

Six hardworking weeks have passed since my last post and now, right before the second evaluation, I am happy to share my last results.

**TL;DR;** I've managed to optimize Word2Vec and achieved fully linear scale using multistream approach. Now, it's **3x** times faster than current Word2Vec in gensim/develop and **2x** faster than original Mikolov's word2vec implementation. See the numbers:

* [gensim/develop version](https://gist.github.com/persiyanov/56f1a162de4645b3ff737f6f53b17c98)
* [Mikolov's version](https://gist.github.com/persiyanov/c945f31ed5ff2b2244f5999be0f7b5bc)
* [New multistream training optimized with Cython](https://gist.github.com/persiyanov/dc171689da36e824781d1e7432bb68bb)

Also, I've optimized vocabulary building using `multiprocessing` module and multistream. See my [pull request](https://github.com/RaRe-Technologies/gensim/pull/2078).


# Plan for the last month

For the last month there is a lot of work to deliver my feature to **develop-branch ready stage**.


See you in the next blogpost! Feel free to reach me via telegram `@persiyanov` or email `dmitry at persiyanov dot gmail dot com`.
