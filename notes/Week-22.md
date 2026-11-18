# Week-22
*9/11/26 - 15/11/26*

## DSA
I started with dsa again in preparation for interview and technical rounds.

### Arrays
I noticed that most of the array related dsa problems require some usage of two pointer method or sometimes even 3 pointer method. The pointer's behaviour depends upon the problem but the pointers themselves are used a lot in arrays.

I also learned that I should always first look if the problem can be solved using only a single pass or will it require multiple passes, I found that this information makes it easier to come up with the logic later.

#### Boyer-Moore Voting algorithm
Finds a majority element in a single pass with two variables of state: a candidate and a count. Walk through the array maintaining a "candidate" value and a counter. When the current element equals the candidate, increment the counter. When it differs, decrement the counter. When the counter reaches zero, the next element becomes the new candidate with a counter of 1. The candidate left at the end is the majority element.
*Phase 2 verification is mandatory when the problem does not guarantee a majority exists.*

n/3 variant maintains two candidates and two counters. The mechanism is cancellation. When an element matches neither candidate and both counters are non-zero, the algorithm decrements both counters, discarding one occurrence of each candidate along with the current element. Three distinct elements remove each other from contention. After the first pass, the two surviving candidates are the only possible answers, but survival alone does not prove either exceeds n/3, so a second pass counts their true frequencies.
 
