+++
title="Euclidean Algorithm"
date=2025-01-09

[taxonomies]
categories = ["Algorithm"]
tags = ["Algorithm","Euclidean"]
+++

While solving LeetCode problems, I ran into a problem called "Greatest Common Divisor of Strings." At first, I tried to solve it by comparing two strings with pointers without thinking too deeply. I soon realized that approach was wrong. As usual, reading and analyzing the problem carefully matters.

The key was to find the largest common string. If I translate that into the lengths of the strings, the problem becomes a greatest common divisor problem.

I had never implemented a greatest common divisor function before, so I tried thinking through it myself for a while. Eventually I looked for help and found this very clean code:

```C#
public int gcd(int a, int b) {
    return (b == 0)? a : gcd(b, a % b);
}
```

That is when I learned this was the famous Euclidean algorithm.

The method I originally knew was to factor both numbers and multiply their common factors.
For example, to find the GCD of 252 and 105: `252 / 3 = 84`, `105 / 3 = 35`, then `84 / 7 = 12`, `35 / 7 = 5`. Since 12 and 5 have no common factor, 3 and 7 are the common factors of 252 and 105, and `3 * 7 = 21` is the greatest common divisor.




References \
<https://www.khanacademy.org/computing/computer-science/cryptography/modarithmetic/a/the-euclidean-algorithm>
<https://seunghyum.github.io/algorithm/Euclidean-algorithm>




