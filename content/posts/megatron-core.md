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


##### References:

* [Lyuwen's Megatron Core code](https://github.com/lyuwen/Megatron-LM/tree/optm_dataset).

