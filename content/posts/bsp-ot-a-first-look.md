+++
date = '2026-08-30T16:16:23+01:00'
draft = false
title = 'Learning Something Today: BSP-OT'
tags = ["Optimal Transport", "Computer Graphics"]
+++

This post is the first (of hopefully many) where I will go through the steps of reading/understanding/implementing a paper I'm reading. Since I am just a mere Technical Artist (and just a Crowds guy on top of that, not even a FX smartass!), that process will probably be quite bumpy but it might be interesting to some, or at least to me, so that I can put my thoughts in order. Expect quite a few rants about conceptual aspects I find beautiful but do not fully grasp yet. Feel free to contact me (or correct me!) any time, you know where to find me -> niconghiem@gmail.com

Today's topic is the BSP OT paper by Genest et al. submitted at SIGGRAPH Asia 2025, that you can find over there:  https://dl.acm.org/doi/10.1145/3763281 Our goal will be to implement a first version of the BSP-OT paper in the bijective case (which corresponds to Section 3.) Please refer to this for the Houdini files (both scene file with rudimentary Houdini native implementation and a HDA using the HDK for speed) https://github.com/NicoNghiem/nn-hda-collection/tree/main/bsp-ot


# BSP OT bijective case

## What are we trying to solve?

Optimal Transport is one of my favorite topics and while I'm quite tempted to go through the Monge-Kantorovich formulation of the problem, I will keep the formulation quite pragmatic here as one of the cases presented in the paper is quite a perfect way to illustrate it and most Houdini artists will see immediately what problem we are trying to solve.

Let's say you have two point clouds of same size and you want to find a mapping between the two of them: 
![Point Clouds to Match](/images/bsp-ot-a-first-look/point_clouds.png)

Here is an example of what we can do if we find such a mapping:
![Point Clouds to Match](/images/bsp-ot-a-first-look/point_clouds_morphing.gif)

On the surface, this problem might sound a little underwhelming (and maybe even easy to solve if you've never tried to solve it in practice) but it is actually quite a challenging problem that is computationally expensive to solve precisely. This belongs to a class of problems called Optimal Transport problems: the goal is to find a transport plan between two weight distributions (in our case, each point can be seen as having a weight of 1/N) There's quite a bunch of applications in the real world, the most infamous one being that this Optimal Transport plan induces a metric useful to measure distances between distributions (even with non-overlapping support!) which you might encounter named as the Wasserstein Distance. For more readings on the topic, I highly recommend Gabriel Peyre and Marco Cuturi's book on Optimal Transport https://arxiv.org/abs/1803.00567 (which I own a hard copy at home -I'm such a fan!)

For the little story, I first encountered Optimal Transport when I was still a research intern at KAIST's VML lab where I needed to find a distance to compare some indicators (tied to Motion Capture Data) that I stubbornly wanted to interpret as measure distributions. I ended up meeting it again a little after I joined Disney Animation, during the production of Raya: I got to talk regularly with Jake Rice about OT (I was more or less his mathematics rubber duck -fascinating experience, 100% recommend) when he was working on his Druun's webbing on Raya that used OT in one of its steps https://jakerice.design/2024/01/18/RayaAndTheLastDragon/

Back in that time, the main algorithm we were using was Sinkhorn iterations (https://papers.nips.cc/paper_files/paper/2013/hash/af21d0c97db2e27e13572cbf59eb343d-Abstract.html) which did not scale really well for high number of points and high precision. The research on the topic advanced quite a bit since then and the BSP-OT algorithm is such a massively convenient tools for the discrete use case that we, CG hobbyists/enthusiasts end up running into quite often!

## A step back: 1D OT and sorting

To understand properly BSP-OT keeping constantly in mind a little fact -that is the basis of Sliced Optimal Transport- is so useful! Sliced OT is a branch of OT where we use slicing to help compute the Wasserstein distance, using one EXTREMELY important fact:

**Computing Optimal Transport in 1D is really easy!!!**

As a matter of fact, it almost boils down to Sorting. To convince yourself of this fact (not really a proper proof but hey!) look at this below:
![1D OT](/images/bsp-ot-a-first-look/1d_ot.png)
We moved the blue distribution slightly just for visibility but the data is indeed 1D because it lives on a line, the mapping can easily be computed by sorting both distributions and mapping in that same order!
Sliced OT uses this fact and, to summarize it really crudely, rather than brute forcing the Wasserstein distance, it projects our distributions on multiple random directions, does blazing fast 1D OT and compute an integral approximation of the Wasserstein distance using Monte Carlo. (See Cuturi's book or the more recent https://arxiv.org/pdf/2508.12519 if you want a less butchered explanation). The thing with Sliced OT -that the BSP OT paper mentions- is while it is great to approximate Wasserstein distances, it is not providing easily a transport plan (what we are after) between our distributions and that's quite annoying to us

On a side note, I always found the relationship between Optimal Transport and sorting quite fascinating (big fan of this paper that uses OT to provide a Differentiable Sorting algorithm: https://arxiv.org/pdf/1905.11885) and you can find this relationship again in the BSP-OT at multiple places!

## BSP OT: bijective case

Let's finally come to the BSP OT algorithm in the bijective case (aka, same number of points in the original and the target geometries). The idea behind it is kind of simple:
- Generate multiple transport plans
- Look at all the transport plans and combine them to get a better one

Eventually, if the original transport plans are not too bad, quick to compute and we have a solid plan to combine them efficiently, we should get a fast and decent result, shouldn't we? Though those conditions are not that obvious and that's where the BSP trees come to play.

### Binary Space Partitioning

BSP stands for Binary Space Partitioning, it is a Divide and Conquer type algorithm: hopefully, you will see how closely it is related to sorting after reading all of this! And if you ever built kd-trees/octrees, this will sound very familiar to you too!
The procedure is as follow:

- We pick a direction randomly, we project all our points along that direction and use this to split each Point Set in two cells, based on that same direction (see below). The idea behind this is that a purple point from the square should eventually be mapped to a purple point of the circle (same with green)
![BSP Step 1](/images/bsp-ot-a-first-look/bsp_step1.png)
- For each Cell, we repeat the same procedure but with a different random direction (can be different for each cell)
![BSP Step 2](/images/bsp-ot-a-first-look/bsp_step2.png)
- We continue until we only have cells containing a single point! The mapping is then obvious: a point is mapped to the point of the corresponding cell in the other Point Set!
![BSP Matching](/images/bsp-ot-a-first-look/bsp_matching.gif)

Speaking (always!) about sorting, notice how picking systematically the same direction would be equivalent of doing a 1D sort!

The Houdini implementation is quite straightforward:
![BSP Pseudo](/images/bsp-ot-a-first-look/bsp_matching_pseudoalgo.png)

To facilitate the For Loop iteration we merge both point clouds in a single stream. Both branches are the exact same procedure to make sure the Point Clouds are splitted the same way. We first compute the dot product against the given direction like this (notice how the cell name is used as a seed to allow constant shuffling of the randomization):
```
// compute dot product
int iteration = detail(1,'iteration');
float cell_rand = random_shash(s@cell_name)*0.2;
float iter_rand = random(iteration);
vector dir = sample_direction_uniform(set(cell_rand, iter_rand));
f@__dot = dot(v@P, dir);
```
Then, once sorted, we can safely break our cells into subcells like this:
```
// find your id within your cell
int ids[] = findattribval(0,'point','cell_name',s@cell_name);
int N_cell = len(ids);
int myId = find(ids,@ptnum);

// Cut through the middle
// the way we're encoding our cells is by naming them using binary numbers
if(myId < floor(0.5 * N_cell))  s@cell_name += '0';
else    s@cell_name += '1';
```

At the end of the day, two points mapped share the same `cell_name`

### Merging Two Transport Plans

Now that we are able to generate multiple plans, the goal is to improve on them by merging them with a procedure. There are two main strategies that are used by the paper to do it:
- the obvious one is by doing greedy swaps: we can compare two assignments, if one is better than the other then we pick that one instead. One catch with this one though: since we need to maintain the bijectivity (one point is mapped to **exactly** one point) doing a swap means in reality comparing the result of 2 swaps
- sometimes, some swaps involve more than just two combinations: the paper notices that we can cluster two graphs into connected components. See those two different transport plans:

![BSP Start](/images/bsp-ot-a-first-look/bsp_matching_start.png)

If you superpose them you will see some connected components appear:
![BSP CC](/images/bsp-ot-a-first-look/bsp_merging_CC.png)

The ones that appear all red are actually just superposed (red and blue) and got the same assignment. However the bigger ones are the ones that involve more work: we will choose the color that works the best and then performs locally greedy swaps to try to improve them. In our scenario, blue is always better but we can imagine other scenarios where some red are better and some blue are better, leading to a global improvement of our plan!

As a result, here is what happens when merging multiple trees: you can see that the overall plan improves quite quickly:
![BSP Merging](/images/bsp-ot-a-first-look/bsp_merging_timelapse.gif)

And that's all! Regarding the Houdini implementation we just have to add a little network under our previous BSP Matching and incorporate everything in a for-loop (for adding and comparing to a new tree sequentially):
![BSP Merging](/images/bsp-ot-a-first-look/houdini_bsp_ot.png)

We first need to identify the Connected Components (SOP Connectivity is great for that so we just have to create the wirings similarly to the figure above), then we run the greedy improvements on those Connected Components and voila!

### Moving forward

The paper proposes multiple ameliorations: better strategies of picking directions (rather than a stupid completely random) and covering the Unbalanced case which is quite more involved. But that's enough for this article as it is already becoming quite long!