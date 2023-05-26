+++
title="704. Binary Search"
date=2023-05-05

[taxonomies]
categories = ["Leet code"]
tags = ["rust","leetcode"]
+++


## Problem
Given an array of integers nums which is sorted in ascending order, and an integer target, write a function to search target in nums. If target exists, then return its index. Otherwise, return -1.

You must write an algorithm with O(log n) runtime complexity.

**Example 1:**
> **Input:** nums = [-1,0,3,5,9,12], target = 9  
> **Output:** 4  
> **Explanation:** 9 exists in nums and its index is 4  

**Example 2:**
> **Input:** nums = [-1,0,3,5,9,12], target = 2  
> **Output:** -1  
> **Explanation:** 2 does not exist in nums so return -1  

**Constraints:**
- 1 <= nums.length <= $10^4$  
- -$10^4$ < nums[i], target < $10^4$  
- All the integers in nums are unique.  
- nums is sorted in ascending order.  


```rust
pub fn search(nums: Vec<i32>,target: i32) -> i32{
    let mut left=0;
    let mut right=nums.len()-1;
    while left<right{
        let med = (left+right)/2;
        if nums[med]==target{
            return med as i32;
        }
        else if nums[med]<target{
            left=med+1;
        }else{
            right=med-1;
        }
    }
    return -1;
}
```