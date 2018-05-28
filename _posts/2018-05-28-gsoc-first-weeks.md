---
layout: post
title: "First two weeks in GSoC, Gensim project"
date: 2018-05-28 14:00:00 +0300
categories: jekyll update
---


# Greetings!

As you know from the my [previous]({{ site.baseurl }}{% post_url 2018-04-24-accepted-to-gsoc-2018 %}) blogpost, I am working on Gensim library as a GSoC student this summer. In this post, I'd like to share my results, discoveries and fails during Week 1 & Week 2. In the end, I will describe my plan for next weeks (which is no longer aligned with original proposal timeline, unfortunately).


## Intro

Initially, the problem I'm trying to solve was sounded like this: threads that train the model (workers) are highly optimized and therefore are blocked on `Queue.get` call [here](https://github.com/RaRe-Technologies/gensim/blob/develop/gensim/models/base_any2vec.py#L90) and wait until data producer threads (producers) fill the queue [here](https://github.com/RaRe-Technologies/gensim/blob/develop/gensim/models/base_any2vec.py#L125).

In order to fix this, *multistream API* was proposed. The main idea is to read data in many threads instead of using single thread (call several [job producers](https://github.com/RaRe-Technologies/gensim/blob/develop/gensim/models/base_any2vec.py#L125) in parallel on different input streams). Then, user could split large data file into several parts and pass them as *input streams* to the model. Sounds great, but we can already see one subtle thing here:

> Python doesn't allow to execute CPU-bound operations in parallel (it uses GIL, which prevents this). Therefore, if our `_job_producer` loop is more CPU-bound than IO-bound, our multistream optimization may not lead to performance boost.

## Single stream benchmarking

**Note**: You can find full results in [my pull request](https://github.com/RaRe-Technologies/gensim/pull/2048#issuecomment-389497616), but in this blog post I will attach results related only to Word2Vec.

I started with benchmarking *single input stream* models performance.

| # workers | total time (sec) | avg queue size | processing speed (words per sec) | sum cpu load (%) |
|:---------:|:----------------:|:--------------:|:--------------------------------:|:----------------:|
|     1     |      1182.58     |      1.48      |              153350              |      104.35      |
|     4     |      313.03      |      7.09      |              579322              |      395.23      |
|     8     |      277.35      |      0.025     |              653863              |      456.55      |
|     10    |      275.24      |      0.040     |              658857              |      456.85      |
|     12    |      285.95      |      0.024     |              634182              |      449.99      |
|     14    |      288.64      |      0.017     |              628288              |      452.17      |


Up to 4 workers everything is okay:
* Approx. linear speedup for 1-to-4 workers transition
* ~4 CPUs are fully utilized

But increasing number of workers to 8, 10, 12, 14 shows the problem with workers starvation:
* Avg queue size is almost zero
* Processing speed doesn't increases linearly, it gets on a plateau.
* CPUs are not fully utilized (each CPU is utilized by ~20-40%)


## Multistream API and word2vec main discovery

I implemented a first version of multistream API (just modified `BaseAny2VecModel._train_epoch` method to [this](https://gist.github.com/persiyanov/0a8ca3d9091775bd136cfe6e4674e376)) one). 

But [benchmarks](https://gist.github.com/persiyanov/1009e2a4548ac71efa59a547d336fe4b) are not as good as they were supposed to be. Moreover, in some cases __multiple streams hurts__ the performance in compare to single stream. This proves that `_job_producer` is actually CPU-bound, not IO-bound. So, when we create many producers, because of GIL, less resources are given to `_worker_loop` threads and performance doesn't increase.

While I was fighting all these multithreading subtleties and resource contention things, I decided to run an experiment which doesn't have any *job producers* at all but only *worker threads*. I wanted to measure maximum word2vec speed (no IO, only CPU model updates) which is an upper bound for my results in this project. In order to do this, I modified the code of `_train_epoch` to fill the queue with all data at first and then run worker threads (see the code [here](https://gist.github.com/persiyanov/bceb706b2d617ebde69e11774fe8dc16)). Here are the results:

| # workers | total time (sec) | processing speed (words per sec) | sum cpu load (%) |
|:---------:|:----------------:|:--------------------------------:|:----------------:|
|     1     |      1047.96     |              173050              |      101.11      |
|     4     |      261.16      |              694407              |      386.01      |
|     8     |      197.32      |              919021              |      545.11      |
|     10    |      197.67      |              917400              |      536.40      |
|     12    |      201.49      |              900022              |      544.90      |
|     14    |      203.63      |              890559              |      546.92      |

__What are these numbers talking about?__ They are saying, that even __clean word2vec can't scale linearly__ after 4 threads. So, we don't have linear speed-up not because of IO problems, mainly because of word2vec throughput capabilities. But, nevertheless, word2vec with inmemory queue boosts up to +300k words per sec, so there is a space for optimizations.


## Plan for Week 3 & Week 4


1. I concluded that `_job_producer` is CPU-bound, so there is a contention between producers and workers. I want to implement multistream using __multiprocessing__, not multithreading (there is no GIL between processed in python) and benchmark it.
2. Many users complain that [vocabulary building](https://github.com/RaRe-Technologies/gensim/blob/develop/gensim/models/base_any2vec.py#L462) stage is slow (see [this](https://github.com/RaRe-Technologies/gensim/pull/2048#issuecomment-389855592) Radim's comment). I will parallelize it using multistream (likely multiple processes, not multiple threads because vocab building is CPU bound).



So guys, stay tuned and come back in few weeks! Feel free to reach me via telegram `@persiyanov` or email `dmitry at persiyanov dot gmail dot com`.