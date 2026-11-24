# Week - 24
*23/11/26 - 29/11/26*


### Hash Tables
It's basically a python dictionary. I used to get confused first when I used to see 'hash map' or 'hash table' but they are all are just a python dictionary as I understand it. It's easier for me to remember them all if i just consider them as different words used to describe a dictionary.

Generally from what i have observed, these are used in problems where we have to count something's occurrence in a string, array, etc. 

#### Encode and Decode tinyURL
I this problem i need to encode a URL into a tiny URL and decode those encoded URLs (tinyURL) back into their original form.
I can do this using a code. I can generate a random 6 digit code for each URL and in a hashmap store each URL and it's assigned code. Then i can make this code into the shortened URL by appending it in front of a common predefined prefix like `http://tinyurl/`. I will need to create another hashmap that will store the code and URL as key and value respectively for decoding, as in decoding i will receive the code from which i will be able to get the original URL by consulting this hashmap.

The actual dsa questions that i will be getting in interviews and dsa rounds are like these where it's not obvious what to use. The problems i have written about till now have been straightforward where the point of the problems were to use those specific methods but in medium and hard problems i will have to figure out new ways to solve them and use these methods in those ways to get the results.

#### Reorganize String
In this problem I have to reorganize a string such that no two adjacent characters are the same. 
The most optimal way is to count the frequencies of each character. Find the most frequent character. Place all of its copies at even indices first, since it needs the most separation. Place every remaining character in the remaining slots, continuing the even-then-odd order. If the most frequent character exceeds `(n+1)/2` then there is no possible answer. 
The main question was how to store and retrieve the character frequencies, The solution said to use an array of size 26. For each character it's index will be `ord(c) - ord('a')` and from the index i can find the character by `chr(i + ord('a'))`. It's a good way which fast and occupies O(1) space but i had the idea of using hashmap which will be a bit slower but will allow me to parse strings with any chars - uppercase, digits, Unicode while the array method only allows for lowercase alphabets.
```
freq = Counter(s)
max_char = max(freq, key=freq.get)
```

#### Longest Consecutive Sequence
Given an unsorted array of integers nums, return the length of the longest consecutive elements sequence. 
Ordinarily this would be a simple problem of sorting then traversing once, but the restriction is that the time complexity must be `O(n)`, since sorting itself will take `O(n log n)` time it's not feasible here.
My first instinct was DP since this looks like a sequential DP problem but when i looked into why DP is not possible I realized that DP needs a topological order. To compute dp[x] I need dp[x+1] first. The natural order is sorted order which is again `O(n log n)`.
I could not see any obvious way to do this so looked at the solution. Turn out we need to use a hash set here. The real fork in the road is: do I sort to get my DP order `(O(n log n))`, or do I get the order for free some other way `(O(n))`? A hash set is the "some other way", it lets  me answer "is x+1 present, and what's its dp?" in O(1) in discovery order.

The idea is to count only from sequence beginnings. A number begins a sequence exactly when `num - 1` is not in the set. First i make a set of the array. Then for each number whose previous number ( i.e. number-1) is not present in the set I count the length of consecutive numbers in the set and store the largest till now in a variable. 

** Rule of thumb: if the recurrence has no max/min over choices — if the next state is forced, you're not really doing DP, you're doing traversal, and the data structure (not the recurrence) is the whole problem. **


#### Number of Good Ways to Split a String
the left and the right part should have equal number of distinct characters and left + right should equal the whole string.

The most straightforward approach would be to parse the string one time from both sides (i.e. from 0 to n-1 and from n-1 to 0) and store the prefix and suffix distinct character count in separate arrays for each index in the string. Then parse the two array at once to find the number of times their values are same.

#### Minimum Deletions to Make Character Frequencies Unique
Minimum number of characters you need to delete so that no two different characters have the same frequency.

The first thing that came to my mind was to make an array of frequencies of distinct characters, since we don't need the actual characters, we can just get the array of numbers. It needs to be then sorted in decreasing order and then I parse through it and for every element equal to previous element i make it 1 less than previous element and add the difference to a count which will be the number of characters to be deleted (i.e. the value to be returned).

Technically I can also do this using a hash set and then reduce the frequency count if same exists in the set and this wont increase the time complexity because of the O(1) lookup of a set, but the first method feels more straightforward.


### Two Pointer

#### Trapping Rain Water
The problem: Given an elevation map height[] (bar width 1), compute how much rain water it traps.

Only the shorter of the two walls ever matters. So for each bar, I don't need to know both walls exactly. I only need to know which side is the shorter one, and then I can use that side's wall alone.

Stand one marker at the far left end and another at the far right end. Keep track of the tallest bar each marker has seen so far on its own side. Then repeat this: compare the bar under the left marker with the bar under the right marker, and deal with the shorter one.

If the left bar is shorter, figure out the water on it using the tallest bar seen so far from the left. Add that water to the total, then move the left marker one step inward. If the right bar is shorter (or equal), do the same thing from the right side. Keep going until the markers meet.

Suppose the left bar is the shorter one. The tallest wall I've seen on the left was passed earlier, at a moment when the bar on the right side was taller than it. Markers only move inward, so that taller bar on the right is still out there, somewhere to the right of my current position. That means the true tallest wall on the right is at least as tall as my left wall. So the left wall is the shorter (or equal) one, and it is the one that sets the water level. I can compute the water with it and not worry about anything else.

- A shorter bar in the middle changes nothing. Adding a smaller bar can never lower the tallest bar on that side.
- A taller bar in the middle does make the true right wall higher, but that only pushes it further above my left wall. Since only the shorter wall counts, the extra height is ignored. And when the left marker eventually walks up to that tall bar, it becomes the new left wall for everything after it.


#### Count Subarrays With Fixed Bounds
Here I have to find the number of subarrays which the minimum value in the subarray is equal to `minK` and the maximum value in the subarray is equal to `maxK`.

I can count the subarrays by their right endpoint i.e. i can count how many valid subarrays end at this index, so i only need to keep track of the left index of the subarrays.
I parse through the array and keep track of the last minimum and maximum value index (minimum and maximum according to the given values `minK` and `maxK`) and I also keep track of the last bad value i.e. a value less than `minK` or greater than `maxK`. 

The subarray in which the last min and last max indices are the extreme indices form the shortest possible valid subarrays and the values between it and the last bad value can also be included in the subarray as they don't make invalid.
So for each index I can calculate `min(lastMin, lastMax) - lastBad`, if it is positive then i add it to the final count.



