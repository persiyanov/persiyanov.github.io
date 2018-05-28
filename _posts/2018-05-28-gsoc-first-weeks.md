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

```
----- MODEL "full-word2vec-window-10-workers-01-size-300" RESULTS -----
* Total time: 1182.58386683 sec.
* Avg queue size: 1.48076923077 elems.
* Processing speed: 153350.576721 words/sec
* Avg CPU loads: 0.06, 0.02, 0.05, 99.40, 1.68, 0.25, 2.48, 0.11, 0.14, 0.03, 0.02, 0.00, 0.00, 0.01, 0.00, 0.10
* Sum CPU load: 104.35

----- MODEL "full-word2vec-window-10-workers-04-size-300" RESULTS -----
* Total time: 313.03660202 sec.
* Avg queue size: 7.09708737864 elems.
* Processing speed: 579322.695268 words/sec
* Avg CPU loads: 0.13, 0.06, 1.03, 0.16, 0.91, 16.28, 0.01, 0.01, 94.11, 22.10, 72.40, 0.40, 0.00, 0.00, 93.96, 93.67
* Sum CPU load: 395.23

----- MODEL "full-word2vec-window-10-workers-08-size-300" RESULTS -----
* Total time: 277.351661921 sec.
* Avg queue size: 0.0255474452555 elems.
* Processing speed: 653863.581506 words/sec
* Avg CPU loads: 25.71, 29.21, 19.85, 24.07, 38.67, 42.76, 27.28, 37.72, 29.91, 29.09, 35.91, 35.46, 21.26, 13.55, 29.03, 17.07
* Sum CPU load: 456.55

----- MODEL "full-word2vec-window-10-workers-10-size-300" RESULTS -----
* Total time: 275.248829842 sec.
* Avg queue size: 0.0404411764706 elems.
* Processing speed: 658857.998068 words/sec
* Avg CPU loads: 25.60, 26.90, 27.83, 21.12, 22.92, 29.67, 35.28, 40.76, 34.92, 33.96, 32.69, 33.75, 34.16, 26.19, 18.26, 12.84
* Sum CPU load: 456.85

----- MODEL "full-word2vec-window-10-workers-12-size-300" RESULTS -----
* Total time: 285.958873987 sec.
* Avg queue size: 0.0247349823322 elems.
* Processing speed: 634182.830109 words/sec
* Avg CPU loads: 23.00, 24.52, 27.39, 26.22, 29.77, 29.84, 37.62, 39.08, 32.67, 32.38, 30.09, 29.49, 27.07, 24.23, 22.69, 13.93
* Sum CPU load: 449.99

----- MODEL "full-word2vec-window-10-workers-14-size-300" RESULTS -----
* Total time: 288.641264915 sec.
* Avg queue size: 0.0175438596491 elems.
* Processing speed: 628288.079506 words/sec
* Avg CPU loads: 22.97, 21.53, 27.90, 26.05, 31.30, 30.67, 35.71, 36.23, 34.16, 30.63, 30.53, 32.36, 27.86, 21.09, 22.16, 21.02
* Sum CPU load: 452.17
```


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

```
----- MODEL "inmemory-word2vec-window-05-workers-01-size-300" RESULTS -----
* Total time: 1047.96227694 sec.
* Processing speed: 173050.499995 words/sec
* Avg CPU loads: 0.35, 0.15, 0.01, 10.18, 0.01, 0.22, 40.98, 0.02, 0.07, 0.08, 0.01, 6.95, 0.02, 0.17, 41.83, 0.05
* Sum CPU load: 101.11139
----- MODEL "inmemory-word2vec-window-05-workers-04-size-300" RESULTS -----
* Total time: 261.160044909 sec.
* Processing speed: 694407.653604 words/sec
* Avg CPU loads: 0.01, 0.00, 91.79, 0.62, 0.01, 0.01, 0.02, 0.00, 96.12, 96.19, 0.00, 0.00, 0.09, 96.15, 0.59, 4.40
* Sum CPU load: 386.01218
----- MODEL "inmemory-word2vec-window-05-workers-08-size-300" RESULTS -----
* Total time: 197.32940197 sec.
* Processing speed: 919021.216249 words/sec
* Avg CPU loads: 42.79, 29.04, 51.25, 46.97, 25.08, 50.11, 24.76, 55.08, 25.72, 38.73, 16.26, 21.44, 42.91, 18.56, 43.74, 12.68
* Sum CPU load: 545.1179
----- MODEL "inmemory-word2vec-window-05-workers-10-size-300" RESULTS -----
* Total time: 197.677942991 sec.
* Processing speed: 917400.845313 words/sec
* Avg CPU loads: 31.71, 31.89, 32.00, 31.07, 27.23, 34.32, 38.53, 40.70, 36.76, 36.67, 33.95, 34.57, 40.22, 32.31, 29.17, 25.31
* Sum CPU load: 536.402
----- MODEL "inmemory-word2vec-window-05-workers-12-size-300" RESULTS -----
* Total time: 201.494498968 sec.
* Processing speed: 900022.084616 words/sec
* Avg CPU loads: 36.45, 30.31, 36.85, 36.21, 36.02, 35.97, 36.08, 37.56, 33.15, 36.31, 31.85, 31.83, 31.19, 33.68, 31.60, 29.84
* Sum CPU load: 544.9081
----- MODEL "inmemory-word2vec-window-05-workers-14-size-300" RESULTS -----
* Total time: 203.636204958 sec.
* Processing speed: 890559.672517 words/sec
* Avg CPU loads: 36.61, 35.04, 34.93, 37.11, 35.00, 35.43, 35.59, 35.62, 32.39, 34.17, 34.72, 31.26, 32.09, 32.48, 30.93, 33.56
* Sum CPU load: 546.9286
```

__What are these numbers talking about?__ They are saying, that even __clean word2vec can't scale linearly__ after 4 threads. So, we don't have linear speed-up not because of IO problems, mainly because of word2vec throughput capabilities. But, nevertheless, word2vec with inmemory queue boosts up to +300k words per sec, so there is a space for optimizations.


## Plan for Week 3 & Week 4


1. I concluded that `_job_producer` is CPU-bound, so there is a contention between producers and workers. I want to implement multistream using __multiprocessing__, not multithreading (there is no GIL between processed in python) and benchmark it.
2. Many users complain that [vocabulary building](https://github.com/RaRe-Technologies/gensim/blob/develop/gensim/models/base_any2vec.py#L462) stage is slow (see [this](https://github.com/RaRe-Technologies/gensim/pull/2048#issuecomment-389855592) Radim's comment). I will parallelize it using multistream (likely multiple processes, not multiple threads because vocab building is CPU bound).



So guys, stay tuned and come back in few weeks! Feel free to reach me via telegram `@persiyanov` or email `dmitry at persiyanov dot gmail dot com`.