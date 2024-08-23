+++
draft = false
date = 2024-08-20
title = "Finding efficiency gains in data loading procedure in Megatron Core framework"
description = ""
slug = ""
authors = ["Lyuwen Fu"]
tags = ["Large Language Models", "AGI", "Neural Network"]
categories = []
externalLink = ""
series = []
math = true
+++

It supprises me that I am able to extract over hundred times efficiency gains in the SOTA LLM framework Megatron Core, but it happened. Here are the details.


#### Distributed dataset builder

The original algorithm in Megatron Core only attempts to build the dataset indices cache at rank 0 with `torch.multiprocessing` parallel,
and than let the rest of the ranks read the cached data.
But at large scale training with hundreds, or even thousands of GPUs, one has even more CPUs at disposal simultaneously.
Assuming a reasonably high performance parallel file system, the IO cost of reading/writing cache files is negligible.
Thus, it's inherently more efficiently to distribute the building processing to the whole cluster and then read the entire cache back
for all ranks.

#### Blended dataset index builder

The main logic of the existing implementation that builds the indices of the
blended dataset starts from the calculation of the error of the expected sample
size of the list of datasets in floating point number $f_{samples}[i] = wt[i] \times n_{tot}$
(calculated from the product of the dataset weight and the total expected
sample size), with respect to the current number of samples *allocated* to the
datasets $n_{curr}[i]$.
$$e_{samples}[i] = f_{samples}[i] - n_{curr}[i]$$
The dataset with the max error gets a increment to its allocation by 1.
Inherently, when the allocation of a dataset is larger than the expected sample size $f_{samples}[i]$,
it will never get any more allocations, since $e_{samples}[i] < 0$.

Therefore, it's obvious we can simplify the algorithm, since before reaching the floor value of the the expected sample size $t_{samples}[i] = floor(f_{samples}[i])$,
the datasets will continue recieving allocations sooner or later, depending the total number of expected samples.
The situation only begins to change when all the datasets have reached the floor value of the expected sample size $n_{curr}[i] == t_{samples}[i]$.
Then based on the original algorithm, the datasets with the largest fractions $r_{samples}[i] = f_{samples}[i] - t_{samples}[i]$
(the remainder of the expected sample size in float modulo 1) will share the remaining allocations and fill the total number of samples $n_{tot}$.

Thus, the first step is to compute $t_{samples}[i]$ for all datasets, and get the current summary $\sum_{i} t_{samples}[i]$, and the remaining allocations
$n_{diff} = n_{tot} - \sum_{i} t_{samples}[i]$, then we increment the $n_{diff}$ datasets with the largest $r_{samples}[i]$, and get the final sample distribution
$n_{samples}[i]$. With the final sample distribution, we can build the indices for the blended dataset, and shuffle them to randomize the sampling procedure. 


#### Asynchronized index shuffle

After some testing with samples at the trillions scale, a new problem arises that the shuffling of arrays at this scale takes an untolerable amount of time,
becoming the new bottle neck of the whole procedure.
But the analysis of the algorithm shows that we only need a randomized indices that maps the original order to the shuffled in constant time.
And the only information needed to create such shuffled indices is the number of samples, which can be predetermined from the training inputs.

Therefore, we can actually compute the randomized indices asynchronously and in advance (for example, we can start the calculation before we start constructing the low-level datasets).
And when we actually need the indices after we have obtained the blended samples from the algorithm described above, thus eleminating the time needed waiting for a random shuffle.


#### Benchmarks

* When testing at 2 billion sample size and 1000 datasets with randomized weights, the improved blending algorithm improves the time costs from 48 minutes to about 18 seconds.

* When testing with 5-trillion-token samples, the whole dataset building procedure improved from over 4 hours to roughly 490 seconds.


##### References:

* [Lyuwen's Megatron Core code](https://github.com/lyuwen/Megatron-LM/tree/optm_dataset).

