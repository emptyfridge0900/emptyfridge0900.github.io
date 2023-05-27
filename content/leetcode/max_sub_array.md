+++
title="53. Maximum Subarray"
date=2020-09-17

[taxonomies]
categories = ["Leet code"]
tags = ["rust","leetcode"]
+++

## Problem
Given an integer array nums, find the contiguous subarray (containing at least one number) which has the largest sum and return its sum.

**Follow up**: If you have figured out the O(n) solution, try coding another solution using the divide and conquer approach, which is more subtle.

**Example 1:**
> **Input:** nums = [-2,1,-3,4,-1,2,1,-5,4]  
> **Output:** 6  
> **Explanation:** [4,-1,2,1] has the largest sum = 6.

**Example 2:**
> **Input:** nums = [1]  
> **Output:** 1

**Example 3:**
> **Input:** nums = [0]  
> **Output:** 0

**Example 4:**
> **Input:** nums = [-1]  
> **Output:** -1

**Example 5:**
> **Input:** nums = [-2147483647]  
> **Output:** -2147483647

**Constraints:**
- 1 <= nums.length <= 2 * 10<sup>4</sup>
- -10<sup>4</sup> <= nums[i] <= 10<sup>4</sup>

## solution
```rust
impl Solution {
    pub fn max_sub_array(nums: Vec<i32>) -> i32 {
        if nums.len()==0{
            return 0
        }
        let mut max:i32=nums[0];
        for i in 0..nums.len(){
            let mut sum:i32=0;
            for j in i..nums.len(){
                sum=sum+nums[j];
                if sum>max{
                    max=sum;
                }
            }
        }
        max
        
    }
}
```
Time complexity is **O(n<sup>2</sup>)**  
<br/>
<br/>


## Other solution

```rust
impl Solution {
    pub fn max_sub_array(nums: Vec<i32>) -> i32 {
        let mut sum: i32 = 0;
        let mut maxSum: i32 = std::i32::MIN;
        for (i, j) in nums.iter().enumerate(){
            sum = sum + nums[i];
            if sum > maxSum {
                maxSum = sum
            }
            if sum < 0 {    
                sum  = 0    
            }              
        }
        
        return maxSum;
    }
}
```
Time complexity is **O(n)** 
