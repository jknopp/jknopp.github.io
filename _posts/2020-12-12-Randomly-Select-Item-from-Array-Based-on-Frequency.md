---
title: Randomly Select Item from Array Based on Frequency
date: 2020-12-12 10:40:00 -0700
description: JavaScript algorithm to perform random item selection from an array based on a frequency array.
categories: 
  - javascript
tags:
  - javascript
  - algorithms
# header:
#    og_image: /assets/images/posts/header/my-cross-platform-development-environment.png
---

While working on a clients project to develop a JavaScript game based around the catching of falling objects, we needed a way to determine the frequency at which objects would fall, in relation to one another. Meaning some "rarer" objects needed to fall less frequently than other "common" objects, while still mainting a random nature to the generation.

Doing some research into algorithms, I found the following example on [generating a random number based on an arbitrary probability distribution](https://www.geeksforgeeks.org/random-number-generator-in-arbitrary-probability-distribution-fashion). While this sounds quite complicated, it's really a pretty simple concept. You have 2 sets of data, 1 is the set you are looking to generate randomly from, the other dataset is arbitrary numbers representing the frequency of each item in the first dataset occuring. The number is arbirary because it only determines frequency in relation to the other numbers in the frequency dataset. Bigger numbers occur more than smaller numbers, and this is exactly what we needed to weight our "rarer" objects to occur less often. 

The article had some good insights into how to accomplish the task at hand, while maintaining O(n) complexity. The following code my JavaScirpt implementation of the algorithm along with an example of how it could be used.

```javascirpt
var test = 1;
```
