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

The article had some good insights into how to accomplish the task at hand, while maintaining O(n) complexity. The following code is my JavaScirpt implementation of the algorithm along with an example of how it could be used.

```javascript
// Utility function to find ceiling of {random} in array[start..end]
function findCeiling(array, random, start, end) {
    let mid;
    while (start < end) {
        mid = Math.floor((start + end) / 2);
        if (random > array[mid]) {
            start = mid + 1;
        } else {
            end = mid;
        }
    }
    return (array[start] >= random) ? start : -1;
}

// Returns a random item from array[]
// according to distribution array
// defined by frequency[].
export const arrayItemArbitraryProbabilityDistribution = (array, frequency) => {
    const size = array.length;
    // Create and fill prefix array
    let prefix = [];
    prefix[0] = frequency[0];
    for (let i = 1; i < size; ++i) {
        prefix[i] = prefix[i - 1] + frequency[i];
    }

    // prefix[n-1] is sum of all frequencies. 
    // Generate a random number with  
    // value from 1 to this sum  
    const random = ((Math.floor((Math.random() * 323567))) % prefix[size - 1]) + 1;
    // Find index of ceiling of {random} in prefix array
    const index = findCeiling(prefix, random, 0, size - 1);
    return array[index];
}
```

And how it might be used:
```javascript
// Define our array of object we want to randomly select from
const fallingItems = 
[{
    id: 1,
    value: 150,
    image: 'rare.png',
    velocity: 5000,
    size: 60
},
{
    id: 2,
    value: 25,
    image: 'average.png',
    velocity: 2000
    size: 110
},
{
    id: 3,
    value: 2,
    image: 'common.png',
    velocity: 900,
    size: 50
}];
// Define the freqeuncy we want each of the fallingItems to be occur at, 
// relative to one another
const frequencyOfFallingItems = [2, 4, 8];

// Randomly get falling item based on frequency array
const itemToQueue =
    arrayItemArbitraryProbabilityDistribution(fallingItems, frequencyOfFallingItems);
```
