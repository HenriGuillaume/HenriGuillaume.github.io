---
layout: post
title: "Optimal group testing"
date: 2024-11-08
---

<head>
    <!-- Other head elements -->
    <script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
    <script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
</head>

# OPTIMAL GROUP TESTING

This notebook is an intereactive explanation of the paper "Optimal Group Testing" (2020) by Amin Coja-Oghlan, Oliver Gebhard, Max Hahn-Klimroth and Philipp Loick. The paper can be found [here](https://proceedings.mlr.press/v125/coja-oghlan20a.html).\
All credit is evidently due to them, this is merely a class project.

## A little context
Let's say that you're a military doctor, charged with testing your whole army for syphilis. Unfortunately, you need to pay, say 50\$ per sample of blood that you test, that could quickly get expensive!\
However, your name is Robert DORFMAN, and you get the idea that you can get the same amount of information without testing each soldier individually, the idea is the following:\
You mix the blood that you drew from your soldiers in *groups*, and you test the mixed blood samples for syphilis, so that a <span style="color: red;">positive</span> result tells you that *at least one* soldier in the group is infected.\
Now imagine that you get really lucky, and that a pool of blood from 5 soldiers tests <span style="color: green;">negative,</span> you know that none of them are infected, and you effectively saved 4 tests! (that's 200$ you can spend on the toyota supra you've been saving for!)

## Problem setup

We are given a population of size $$n$$, and a number $$k$$ of infected or "defective" individuals. (Note that the term "defective" isn't meant to be reductive of ill people, group testing has applications in manufacturing of certain components).\
We assume that
$$k \sim n^\theta$$
meaning that the number of infected $$k$$ is asymptotically proportional to $$n^\theta$$, for a certain $$\theta \in (0,1)$$ i.e. proportional for a large population.\
You might ask the question:

<center>"Why is the proportion written as an exponent, instead of simply being a multiplicative constant?"</center>

<br>
This has to do with the fact that expressing the number of infected as $$n^\theta$$ makes it grow *sublinearly* with population size, which is more realistic in certain use cases. This expression is also more relevant when we talk about asymptotic results.

The whole point of what is coming is to answer the most basic classification problem in medicine:
<center><font size="6"> Who's sick? </font></center>

In more precise terms, given our $$n$$ individuals $$\{x_1,..., x_n \}$$, we have an unknown vector comprised of $$n-k$$ zero-entries for healthy patients, and $$k$$ one-entries for sick patients. We will denote this vector $$\sigma \in \{0,1\}^n$$. When we say a vector of Hamming weight $$k$$, we simply mean that there are $$k$$ non-zero entries.\
This gives us two goals:
- Get an estimator $$\hat{\sigma}$$ of $$\sigma $$ that suceeds with high probability (*w.h.p.*)
- Minimise the number of tests $$m$$, and importantly find out how low we can make it

There are two types of group testing:

### Non adaptive group testing:
In this variant of the problem, we only have the right to one round of tests. This problem is relevant in practical applications as test results may take time to come, so doing $$R$$ rounds of testing multiplies your waiting time by $$R$$.

### Adaptive group testing:
In this variant, we allow ourselves to do a first round, wait for the results, and then adapt our strategy to the acquired results. This is obviously advantageous over non adaptive testing as our choice of groups can be made more precise thanks to additional information.

## First ideas

### The random bipartite design, Definitive Defectives
In this test design, the individuals $$V = \{x_1, ..., x_n\}$$ partake in $$\Delta$$ tests, chosen uniformly at random among our $$m$$ tests $$F = \{a_1, ..., a_m\}$$. This is represented by a bipartite graph. This design is based upon two simple ideas:
1. Any patient $$x$$ that appears in a *<span style="color: green;">negative*</span> test $$a$$ is <span style="color: green;">negative
2</span>. If a patient $$x$$ appears in a <span style="color: red;">positive</span> test $$a$$, and all other participants in $$a$$ have been marked as uninfected during step 1, then $$x$$ is <span style="color: red;">positive</span>. This is because x being postitive is the "only explaination" for $$a$$ being <span style="color: red;">positive</span>.

<div style="text-align: center;">
    <video src="/assets/SPIV/DD_total.mp4" style="width: 100%;" controls autoplay muted loop></video>
    <p style="font-style: italic; color: #555; max-width: 100%; margin-top: 10px;">A very simple example of inference using definite defectives, diagnosing 4 patients with only 3 tests.</p>
</div>

Obviously, this comes with a few caveats. As you can see, the fact that the test above works relies heavily on the fact that all three healthy patients join a test that the infected patient doesnt.\
Therefore, if we have too few tests, too big a population, or a great proportion of infected, the algorithm fails. This is because we are at a greater risk of infected patients "poisoning the well" by making our preciously informative <span style="color: green;">negative tests</span> <span style="color: red;">positive</span>.

<div style="text-align: center;">
    <video src="/assets/SPIV/DD_limitations.mp4" style="width: 100%;" controls autoplay muted loop></video>
    <p style="font-style: italic; color: #555; max-width: 100%; margin-top: 10px;">"Poisoning the well", by reducing the number of tests and having \(x_1\) make both of our remaning tests <span style="color: red;">positive</span>, therefore containing very little information.</p>
</div>

It is a good idea to pause and ponder about the functioning of definite defectives before moving on to the next part.

## SPIV

This algorithm is the main topic if this text. SPIV, or **Spatial Inference Vertex Cover** is an non-adaptive algorithm that allows us to infer $$\sigma$$ w.h.p. with a minimum number of tests $$m_{SPIV}$$ that is asymptotically better than the number $$m_{DD}$$ that definite defective requires. Please not that the key word here is **asymptotically**. We will come back to this later.\
In the following, we will discuss and try to illsutrate the different stages SPIV, as described in the paper.\
I will ask you to not get frightened or stuck on the mathematical details, and try to focus on the ideas of the algortihm.
### A little intuition
Let's start with what I think are good insights to motivate this design.
1. **It makes sense to diagnose _locally_:** An algorithm like definite defectives connects anyone to anyone else at random. When dealing with a very large population, this doesn't seem very realistic, and one might be tempted to "divide and conquer" the problem.
2. **The problem _isn't linear in nature_:** With the previous insight in mind, you might think something along the lines of "let's divide our population of size $$n$$ into 5 groups then!". However, as $$n$$ grows, you will obvioulsy encounter the same problem as before, so your denominator has to depend on $$n$$. It also has to grow slower than $$n$$ (why?), $$\log(n)$$ is a rather natural choice.
3. **It makes sense to _link compartments_:** Imagine you have divided all of the individuals in one of your subgroups. Having this subgroup cut off from the rest of the population would be a waste of information, hence the idea of linking them.

*Nota bene*: when we say that tests (respectively patients) are "linked", we simply mean that they share patients (respectively tests) in common. 

### Phase 0: Setting up the ring
As the title says, the algorithm relies on a *ring structure*. No algebra there, it simply means that we divide our patients and our tests into compartments and that patients share tests with other patients from following compartments, hence the last compartment shares tests with the first compartments (so that it is also linked), and we obtain a ring.

<div style="text-align: center;">
    <video src="/assets/SPIV/ring_creation.mp4" style="width: 100%;" controls autoplay muted loop></video>
    <p style="font-style: italic; color: #555; max-width: 100%; margin-top: 10px;">The SPIV ring structure, keep in mind that the number of tests, test compartments and the links do not have the value SPIV would give them. In fact, for this few patients, SPIV often gives more tests than patients.</p>
</div>

Here are the constants that we need to determine the ring:
* $$\mathbf{l}$$: The number of compartments our patients and tests are divided into (leaving population compartments of size $$\frac{n}{l}$$)
* $$\mathbf{F[1], ... ,F[l]}$$: The test compartments (partition of $$\{a_1, ... , a_{m_{SPIV}}\}$$)
* $$\mathbf{V[1], ... ,V[l]}$$: The patient compartments (partition of $$\{x_1, ... , x_{n}\}$$)
* $$\mathbf{s, \Delta}$$: A patient in $$F[i]$$  can (and must) choose $$\frac{\Delta}{s}$$ tests in each of the compartments $$F[i+1], ..., F[i+s-1]$$. These tests are chosen uniformly at random *with replacement*. This means a patient can join the same test several time. More on that later.
* $$\mathbf{m, m_{add}}$$: The number of tests and *additional tests*, we will explain the reason for these later. This gives us compartments of size $$\frac{m}{l}$$ and one additional compartment $$\mathbf{F[0]}$$ of additional tests.

From there, we will try and build this algorithm by "[induction](https://en.m.wikipedia.org/wiki/Mathematical_induction)". That is we will answer the two questions:
1. How can we use the information we have about a compartment to better diagnose other compartments? (heredity)
2. How do we diagnose the first compartment to start the process? (pushing the first domino, *a.k.a.* initialization)


### Heredity
Suppose that we have diagnosed all compartments $$V[1], ..., V[i]$$, how do we diagnose $$V[i+1]$$?\
As earlier, patients who have taken a <span style="color: green;">negative test</span>, must be <span style="color: green;">negative.</span> We cannot say anything more about uninfected patients.\
Our next task is to differentiate between <span style="color: red;">positive</span> and <span style="color: green;">negative patients</span> that partook in a <span style="color: red;">positive</span> test, respectively true postives and false postitives. The paper offers us a way to do so, that goes as follows: remember how we diagnosed <span style="color: red;">positive</span> patients in DD by finding those who were "the only explaination" for a test being <span style="color: red;">positive</span>? Well the idea here is similar: For a patient $$x$$ in $$V[i+1]$$, we look in the $$s$$ next test compartments (including $$F[i+1]$$), and we count the number of tests that contain $$x$$ but "hadn't been explained up until now".
Thus we define: $$\mathbf{W_{x,j}}$$ as the number of test in $$F[i+j]$$ that do not contain an infected individual from $$V[1], ..., V[i]$$

![image](/assets/SPIV/count.png)
<div style="text-align: center;">
    <p style="font-style: italic; color: #555; max-width: 100%; margin-top: 10px;">Computing \(W\) for \(x_0^{i+1}\), we can see that in compartment \(F[i+1]\), it accounts for one test that wasnt explained before. However, in compartment \(F[i+2]\), even though it does belong to a <span style="color: red;">positive</span> test, this test is already explained by \(x_i^2\), and therefore gives us no information about \(x_0^{i+1}\), which could very well be <span style="color: green;">negative.</span></p>
</div>

As illustrated above, you can see that if $$x \in V[i+1]$$ often belongs to tests that had no previous explaination, it is likely to be infected. We therefore look at the quantity $$\mathbb{E}[W_{x,j}]$$ to infer the state of $$x$$.\
However, we can (and in fact must) do a little better. Imagine that we are looking at $$F[i+1]$$. Because compartments are only linked to tests *farther* along the ring, and $$V[1], ..., V[i]$$ are not responsible for making $$F[i+1]$$ <span style="color: red;">positive</span>,  there's this idea that $$V[i+1]$$ is the "last shot" at explaining why $$F[i+1]$$ is <span style="color: red;">positive</span>.

<div style="text-align: center;">
    <video src="/assets/SPIV/lastshot.mp4" style="width: 100%;" controls autoplay muted loop></video>
    <p style="font-style: italic; color: #555; max-width: 100%; margin-top: 10px;">We can see that \(x_0^{i+1}\) is the only remaining explaination for \(a_0^{i+1}\) being <span style="color: red;">positive</span>. It therefore has to be <span style="color: red;">positive</span>.</p>
</div>

Conversely, if we look at a <span style="color: red;">positive</span> test $$a \in F[i+s]$$, even though it is connected to $$x$$, the patient making $$a$$ postive could still reside in the next compartments $$V[i+2], ..., V[i+s]$$.

<div style="text-align: center;">
    <video src="/assets/SPIV/firstshot.mp4" style="width: 100%;" controls autoplay muted loop></video>
    <p style="font-style: italic; color: #555; max-width: 100%; margin-top: 10px;">Sure, \(x_0^{i+1}\) could be the reason \(a_0^{i+3}\) is <span style="color: red;">positive</span>, but because we know nothing about the other patients that have taken \(a_0^{i+3}\) (anyone of which could be the culprit), we cannot know if \(x_0^{i+1}\) is responsible.</p>
</div>
This observation leads us to the idea of assigning weights $$\mathbf{w_j}$$ to each $$W_{x,j}$$, and we define $$W_x^* = \sum_{j=1}^{s-1}w_jW_{x,j}$$ We then define a threshold for this value under which we are confident that $$x$$ is <span style="color: green;">negative,</span> this is done by optimising the weights $$w_j$$ that minimize the probability of false <span style="color: red;">positive</span>s being classified as true <span style="color: red;">positive</span> (see the paper for more detailed math).

#### Heredity: Cleanup phase
This step is all about correcting errors commited in the previous phase. For any $$x$$ (that is not in the first $$s$$ compartments as we will see below), we count the number of tests $$a$$ containing $$x$$ that do not have any other explaination. If there are enough such tests (more than $$\ln^{\frac{1}{4}}(n)$$), we infer that $$x$$ is <span style="color: red;">positive</span>. Otherwise we say that $$x$$ is healthy. We repeat this process $$\lceil \ln(n) \rceil$$ times. The idea is that we gave each patients enough tests so that a postitive patient should have at least one test that they don't share with any other <span style="color: red;">positive</span> patients.

### Initialization
We now know how to diagnose $$V[i+1]$$ once we have diagnosed $$V[1], ..., V[i]$$, but this requires stqrting somewhere right? Well yes, and there's really no tricks to getting the algorithm started: the idea is simply to use the "additional" compartment $$F[0]$$ that we defined earlier to run definite defectives on the first $$s$$ compartments. This ensures that the first $$s$$ compartments are diagnose with high probability. We only use it on a small subset of our population, so we still get to keep the lower asymptotic number of tests needed by SPIV. Remember how we said we'd define what we mean by asymptotic earlier? Well now is the time.

## Discussion
As we said earlier, the minimum number of tests $$m_{SPIV}$$ is asymptotically better than the number $$m_{DD}$$ that definite defective requires. Actually, even better than that *it is as close as it can possibly get*. This means that no matter, the proportion, for $$n$$ great enough, we get  $$m_{SPIV} < m_{DD}$$, however, it can be wise to look at what we mean by "great enough". To get a feel for what this convergence rate might look like, I invite you to play around with the graph down below by adjusting the slider for $$t = \theta$$. Try to figure out how many trials it takes to do better than definite defectives, or even to reach the theoretical limit for the required minimum number of tests.

<div style="text-align: center;">
  <iframe src="https://www.desmos.com/calculator/ztbww89dcw" width="100%" height="500" style="border: 1px solid #ccc" frameborder="0"></iframe>
  <p style="font-style: italic; color: #555; max-width: 100%; margin-top: 10px;">The \(x\)-coordinates correspond to our "proportion" of infected \(\theta\), the red curve is the information-theoretic limit, the green curve is \(m_{DD}\), and the blue curve is \(m_{SPIV}\)</p>
</div>

This is not to undermine the greatness of this algortihm though. The fact that the authors managed to reach the theoretical limit $$m_{inf}$$ is beautiful on it's own. But seeing the convergence rate, the topic is, in my opinion, far from being fully explored.\
I hope that you enjoyed reading this post, and that you learnt something from it!