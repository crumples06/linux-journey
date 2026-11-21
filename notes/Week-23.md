# Week - 23
*16/11/26 - 22/11/26*

## DSA

### Strings
Strings related dsa problems also generally involve two pointers. 

#### KMP algorithm
It is a string search algorithm designed to efficiently perform pattern matching. 


### Bit Manipulation

To convert it into binary, we repeatedly divide the number by two and record the remainder. We continue until the quotient becomes zero, then read the remainders from bottom to top.

From what I have observed, there is not any obvious hints in problems that tells to use bit wise operations to solve them. So, it's likely going to be a matter or recognizing problems where these operations be used.

One example is a problem where we are given a non-empty array of integers, every element appears twice except for one and we have to find that single one. Here it's not obvious that I can use bitwise operations to solve this. 
A property is XOR is that any number XOR'd with itself becomes 0, and any number XOR'd with 0 stays the same. So I can use this here. I can XOR all elements together. Because order is irrelevant, the two copies of each paired number sit next to each other in effect and cancel to 0. The single number XORs with 0 and survives.

Given an array nums containing n distinct numbers in the range [0, n], return the only number in the range that is missing from the array. Here also XOR can be used. I can XOR every index from 0 to n together with every value in the array. Each number that appears in both the index range and the array is XOR-ed twice and cancels to zero. The missing number appears only in the range, so it is XOR-ed once and remains as the final result.

I can see that if I can find 2 sets of numbers and one of them is missing a number that is in the other then i can use XOR to find that missing number. If I see such a situation in a problem I can assume that XOR can be used here.

#### Bitwise Right Shift
Shifts the bits of the number to the right and fills the vacated bit positions on the left with 0s (or with 1s for negative numbers). It gives similar effect as of dividing the number with some power of two.
In python the code would be `a >> 1` to right shift by 1.

#### Bitwise Left Shift
Shifts the bits of the number to the left and fills 0 on voids right as a result. Similar effect as of multiplying the number with some power of two.
`a << 1`

#### Counting Bits
I have to return an array where it's `ith` position contains the number of 1 bits in the number `i`. This problem is more about DP than bit manipulation. 
Right-shifting a number by 1 drops its last bit and leaves every other bit unchanged. Since `i >> 1` is always less than `i`, by the time we compute `ans[i]`, the value `ans[i >> 1]` is already in the array. So I can just take the number of bits of `i >> 1` and add 1 to it if the rightmost bit of `i` is 1 else just use that number. 
`ans[i] = ans[i >> 1] + (i & 1)`


#### Power of two
Subtracting 1 from any positive number turns off its lowest set bit and turns on every bit below it. A power of two has exactly one set bit, so `n - 1` flips that single bit to 0 and sets all lower bits to 1, leaving no bit in common with n. 
For a number with more than one set bit, the subtraction only affects the lowest set bit and below; the higher set bits stay in place, so `n & (n - 1)` keeps at least one bit and is non-zero. The expression `n & (n - 1)` therefore equals 0 exactly when n has a single set bit.


### Hash Tables
It's basically a python dictionary. I used to get confused first when I used to see 'hash map' or 'hash table' but they are all are just a python dictionary as I understand it. It's easier for me to remember them all if i just consider them as different words used to describe a dictionary.

Generally from what i have observed, these are used in problems where we have to count something's occurrence in a string, array, etc. 









