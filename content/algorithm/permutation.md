+++
title="Permutation"
date=2023-05-29

[taxonomies]
categories = ["Algorithm"]
tags = ["math"]
+++


While solving LeetCode 441, "Arranging Coins," I saw the formula for adding numbers from 1 to `n`.
Most people have probably seen the school problem where you add the numbers from 1 to 100. It is the formula Gauss supposedly came up with when he was ten years old.
The sum from 1 to `n` can be solved with `n(n+1)/2`. There are already many explanations online, so I do not need to repeat the full proof here.
But the formula felt familiar, and then I suddenly remembered learning `n(n-1)/2`, so I looked it up again.

## $\frac{n(n-1)}{2}$: possible pairs from `n` items

When I searched for it, I found explanations like "total number of subarrays" or "the number of pairs you can form from an *n* element set." [Link](https://math.stackexchange.com/questions/2214839/exactly-how-does-the-equation-nn-1-2-determine-the-number-of-pairs-of-a-given)

| | A | B | C | D | E   |
| --- |---|---|---|---|---|
| A |  | AB | AC | AD | AE | 
| B | BA |  | BC | BD | BE |
| C | CA | CB |  | CD | CE |
| D | DA | DB | DC |  | DE |
| E | EA | EB | EC | ED |  |

This is the handshake problem. Suppose `n` people each shake hands without duplication.
One person has to shake hands with `n - 1` other people, excluding themself.
Then is the total number of handshakes `n(n-1)`?
No, because `n(n-1)` includes duplicates: AB-BA, AC-CA, AD-DA, AE-EA, and so on.





# Permutation

While studying combinations, I ended up reviewing permutations too.
I learned about permutations while solving problems a while ago, but I forgot most of it. So I am studying it again.
Permutations and combinations still confuse me sometimes.

## $n^r$: choose `r` from `n`, with repetition, where order matters

Suppose we have A, B, and C. If we choose 3 items while allowing repetition and considering order, we get:

AAA, AAB, AAC  
ABA, ABB, ABC  
ACA, ACB, ACC  
BAA, BAB, BAC  
BBA, BBB, BBC  
BCA, BCB, BCC  
CAA, CAB, CAC  
CBA, CBB, CBC  
CCA, CCB, CCC  



## $\frac{n!}{(n-r)!}$: choose `r` from `n`, without repetition, where order matters

Because repetition is not allowed, each pick leaves one fewer choice than before.
If we choose 3 balls from 16 billiard balls, the first pick has 16 choices, the second has 15 choices, and the third has 14 choices. That gives `16 x 15 x 14 = 3360` permutations.
To generalize `16 x 15 x 14`, we use factorials.

$\frac{16 * 15 * 14 * 13 * 12 * 11..3 * 2 * 1}{13 * 12 * 11 ... 3 * 2 * 1}$ => $\frac{n!}{(n-r)!}$

## Combination
## $\frac{n!}{r!(n-r)!}$: combinations, removing order from permutations

ABC, CBA, and BCA are all made from the same A, B, and C. If order does not matter, they are all the same combination.

Above, $\frac{n!}{(n-r)!}$ gave us permutations where repetition is not allowed and order matters. If order no longer matters, that becomes a combination.

Back to the billiard ball example. We had `16 x 15 x 14 = 3360` permutations. To get combinations, divide by the number of possible arrangements of the selected items.
How many ways can 3 balls be arranged? `3!`. Four balls can be arranged in `4!` ways. In general, `r` items can be arranged in `r!` ways.

So if we divide 3360 permutations by `3!`, we get 560 combinations.
