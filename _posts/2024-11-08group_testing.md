---
layout: post
title: "Farkas' lemma"
date: 2024-10-31
---

# OPTIMAL GROUP TESTING

This notebook is an intereactive explanation of the paper "Optimal Group Testing" (2020) by Amin Coja-Oghlan, Oliver Gebhard, Max Hahn-Klimroth and Philipp Loick. The paper can be found here: https://proceedings.mlr.press/v125/coja-oghlan20a.html \
All credit is evidently due to them, this is merely a class project.

## A little context
Let's say that you're a military doctor, charged with testing your whole army for syphilis. Unfortunately, you need to pay, say 50\$ per sample of blood that you test, that could quickly get expensive!\
However, your name is Robert DORFMAN, and you get the idea that you can get the same amount of information without testing each soldier individually, the idea is the following:\
You mix the blood that you drew from your soldiers in *groups*, and you test the mixed blood samples for syphilis, so that a positive result tells you that *at least one* soldier in the group is infected.\
Now imagine that you get really lucky, and that a pool of blood from 5 soldiers tests negative, you know that none of them are infected, and you effectively saved 4 tests! (that's 200$ you can spend on the toyota supra you've been saving for!)

## Problem setup

We are given a population of size $n$, and a number $k$ of infected or "defective" individuals. (Note that the term "defective" isn't meant to be reductive of ill people, group testing has applications in manufacturing of certain components).\
We assume that
$$k \sim n^\theta$$
meaning that the number of infected $k$ is asymptotically proportional to $n^\theta$, for a certain $\theta \in (0,1)$ i.e. proportional for a large population.\
You might ask the question:

<center>"Why is the proportion written as an exponent, instead of simply being a multiplicative constant?"</center>

<br>
This has to do with the fact that expressing the number of infected as $n^\theta$ makes it grow *sublinearly* with population size, which is more realistic in more use cases.

The whole point of what is coming is to answer the most basic classification problem in medicine:
<center><font size="6"> Who's sick? </font></center>

In more precise terms, given our $n$ individuals $\{x_1,..., x_n \}$, we have an unknown vector comprised of $n-k$ zero-entries for healthy patients, and $k$ one-entries for sick patients. We will denote this vector $\sigma \in \{0,1\}^n$. When we say a vector of Hamming weight $k$, we simply mean that there are $k$ non-zero entries.\
This gives us two goals:
- Get an estimator $\hat{\sigma}$ of $\sigma $ that suceeds with high probability (*w.h.p.*)
- Minimise the number of tests $m$, and importantly find out how low we can make it

There are two types of group testing:

### Non adaptive group testing:
In this variant of the problem, we only have the right to one round of tests. This problem is relevant in practical applications as test results may take time to come, so doing $R$ rounds of testing multiplies your waiting time by $R$.

### Adaptive group testing:
In this variant, we allow ourselves to do a first round, wait for the results, and then adapt our strategy to the acquired results. This is obviously advantageous over non adaptive testing as our choice of groups can be made more precise thanks to additional information.

## First ideas

### The random bipartite design, Definitive Defectives
In this test design, the individuals $V = \{x_1, ..., x_n\}$ partake in $\Delta$ tests, chosen uniformly at random among our $m$ tests $F = \{a_1, ..., a_m\}$. This is represented by a bipartite graph. This design is based upon two simple ideas:
1. Any patient $x$ that appears in a *negative* test $a$ is negative
2. If a patient $x$ appears in a positive test $a$, and all other participants in $a$ have been marked as uninfected during step 1, then $x$ is positive. This is because x being postitive is the "only explaination" for $a$ being positive.


<div class="video-container">
    <video src="assets/SPIV/output.mp4" controls autoplay muted loop playsinline></video>
</div>

## SPIV

This algorithm is the main topic if this text. SPIV, or **Spatial Inference Vertex Cover** is an non-adaptive algorithm that allows us to infer $\sigma$ w.h.p. with a minimum number of tests $m_{SPIV}$ that is asymptotically better than the number $m_{DD}$ that definite defective requires. Please not that the key word here is **asymptotically**. We will come back to this later.\
In the following, we will discuss and try to illsutrate the different stages SPIV, as described in the paper.\
I will ask you to not get frightened or stuck on the mathematical details, and try to focus on the ideas of the algortihm.
### A little intuition
Let's start with what I think are good insights to motivate this design.
1. **It makes sense to diagnose _locally_:** An algorithm like definite defectives connects anyone to anyone else at random. When dealing with a very large population, this doesn't seem very realistic, and one might be tempted to "divide and conquer" the problem.
2. **The problem _isn't linear in nature_:** With the previous insight in mind, you might think something along the lines of "let's divide our population of size $n$ into 5 groups then!". However, as $n$ grows, you will obvioulsy encounter the same problem as before, so your denominator has to depend on $n$. It also has to grow slower than $n$ (why?), $\log(n)$ is a rather natural choice.
3. **It makes sense to _link compartments_:** Imagine you have divided all of the individuals in one of your subgroups. Having this subgroup cut off from the rest of the population would be a waste of information, hence the idea of linking them.

*Nota bene*: when we say that tests (respectively patients) are "linked", we simply mean that they share patients (respectively tests) in common. 

### Phase 0: Setting up the ring
As the title says, the algorithm relies on a *ring structure*. No algebra there, it simply means that we divide our patients and our tests into compartments and that patients share tests with other patients from following compartments, hence the last compartment shares tests with the first compartments (so that it is also linked), and we obtain a ring.

_ILLUSTRATiON OF THE RING HERE_

Here are the constants that we need to determine the ring:
* $\bold{l}$: The number of compartments our patients and tests are divided into (leaving population compartments of size $\frac{n}{l}$)
* $\bold{F[1], ... ,F[l]}$: The test compartments (partition of $\{a_1, ... , a_{m_{SPIV}}\}$)
* $\bold{V[1], ... ,V[l]}$: The patient compartments (partition of $\{x_1, ... , x_{n}\}$)
* $\bold{s, \Delta}$: A patient in $F[i]$  can (and must) choose $\frac{\Delta}{s}$ tests in each of the compartments $F[i+1], ..., F[i+s-1]$. These tests are chosen uniformly at random *with replacement*. This means a patient can join the same test several time. More on that later.
* $\bold{m, m_{add}}$: The number of tests and *additional tests*, we will explain the reason for these later. This gives us compartments of size $\frac{m}{l}$ and one additional compartment $\bold{F[0]}$ of additional tests.

From there, we will try and build this algorithm by "[induction](https://en.m.wikipedia.org/wiki/Mathematical_induction)". That is we will answer the two questions:
1. How can we use the information we have about a compartment to better diagnose other compartments? (heredity)
2. How do we diagnose the first compartment to start the process? (pushing the first domino, *a.k.a.* initialization)


### Heredity
Suppose that we have diagnosed all compartments $V[1], ..., V[i]$, how do we diagnose $V[i+1]$?\
As earlier, patients who have taken a negative test, must be negative. We cannot say anything more about uninfected patients.\
Our next task is to differentiate between positive and negative patients that partook in a positive test, respectively true postives and false postitives. The paper offers us a way to do so, that goes as follows: remember how we diagnosed positive patients in DD by finding those who were "the only explaination" for a test being positive? Well the idea here is similar: For a patient $x$ in $V[i+1]$, we look in the $s$ next test compartments (including $F[i+1]$), and we count the number of tests that contain $x$ but "hadn't been explained up until now".
Thus we define: $\bold{W_{x,j}}$ as the number of test in $F[i+j]$ that do not contain an infected individual from $V[1], ..., V[i]$

_ILLUSTRATION OF THE COUNT_

As illustrated above, you can see that if $x \in V[i+1]$ often belongs to tests that had no previous explaination, it is likely to be infected. We therefore look at the quantity $\mathbb{E}[W_{x,j}]$ to infer the state of $x$.\
However, we can (and in fact must) do a little better. Imagine that we are looking at $F[i+1]$. Because compartments are only linked to tests *farther* along the ring, and $V[1], ..., V[i]$ are not responsible for making $F[i+1]$ positive,  there's this idea that $V[i+1]$ is the "last shot" at explaining why $F[i+1]$ is positive.
_INCLUDE LAST SHOT ILUSTRATION_
Conversely, if we look at a positive test $a \in F[i+s]$, even though it is connected to $x$, the patient making $a$ postive could still reside in the next compartments $V[i+2], ..., V[i+s]$.
_INCLUDE NOT LAST SHOT ILUSTRATION_
This observation leads us to the idea of assigning weights $\bold{w_j}$ to each $W_{x,j}$, and we define $W_x^* = \sum_{j=1}^{s-1}w_jW_{x,j}$ We then define a threshold for this value under which we are confident that $x$ is negative, this is done by optimising the weights $w_j$ that minimize the probability of false positives being classified as true positive (see the paper for more detailed math).
