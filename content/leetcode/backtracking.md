+++
title="Back tracking"
date=2025-07-23

[taxonomies]
categories = ["Leet code"]
tags = ["rust","leetcode","Backtracking"]
+++

요즘에는 back tracking 문제를 좀 풀어보고 있다. 쉬운것 같으면서도 쉽지가 않다. 머리로 생각한 것을 구현하는 능력도 필요하지만 더 필요한 것은 문제 자체를 해결하는 능력이다.
아직도 많은 연습이 필요하다. 비슷한 문제를 20개 정도 풀어보면 자신감을 얻을 수 있을 거 같다. 
## 78. Subsets
### 문제
Given an integer array nums of unique elements, return all possible subsets (the power set).

The solution set must not contain duplicate subsets. Return the solution in any order.
### Example 1
```
Input:
    [1,2,3]
Output:
[
    [],
    [1],
    [2],[1,2],
    [3],[1,3],[2,3],[1,2,3]
]
```
흠.... 뭔가 패턴이 보이는거 같은데?
```
[
    0:[],
    1:[1], //이전의 리턴값 각 element에 1을 추가함.
    2:[2],[1,2], //이전 리턴값 각 element에 2를 추가함.
    3:[3],[1,3],[2,3],[1,2,3] //이전 리턴값 각 element에 3을 추가함.
]
```

>[] = []  
>[][1] = []은 윗줄에서 copy, 그리고 []에 1삽입  
>[][1] [2][1,2] = [][1]은 윗줄에서 copy, 그리고 [][1]에 2를 삽입해서 [2][1,2]  
>[][1][2][1,2] [3][1,3][2,3][1,2,3서 = [][1][2][1,2]를 윗줄에서 copy, 그리고 [][1][2][1,2]에 3을 삽입해서 [3][1,3][2,3][1,2,3]  

비록 시간이 걸리기는 했지만 이 문제는 그래도 혼자 생각하고 혼자 풀 수 있었다.

### Solution
```rs
pub fn subsets(nums: Vec<i32>) -> Vec<Vec<i32>> {
    let mut ret = vec![vec![]];
    for n in nums.iter(){
        for r in 0..ret.len(){
            let mut x = ret[r].clone();
            x.push(*n);
            ret.push(x);
        }
    }
    ret
}
```

## 39. Combination Sum
### 문제
Given an array of distinct integers candidates and a target integer target, return a list of all unique combinations of candidates where the chosen numbers sum to target. You may return the combinations in any order.

The same number may be chosen from candidates an unlimited number of times. Two combinations are unique if the frequency of at least one of the chosen numbers is different.

The test cases are generated such that the number of unique combinations that sum up to target is less than 150 combinations for the given input.

### Example 1
```
Input: candidates = [2,3,6,7], target = 7
Output: [[2,2,3],[7]]
Explanation:
2 and 3 are candidates, and 2 + 2 + 3 = 7. Note that 2 can be used multiple times.
7 is a candidate, and 7 = 7.
These are the only two combinations.
```
### Example 2
```
Input: candidates = [2,3,5], target = 8
Output: [[2,2,2,2],[2,3,3],[3,5]]
```
주어진 숫자를 **중복**해서 사용할 수 있고 **배열하는 순서가 상관있다**, 즉 [2,3,5] 와 [2,5,3] 은 같은걸로 친다는 말이다.


Input이 [2,3,6,7] 라고 할때 처음 시도 해볼 수 있는 방법은 DFS 방법으로 하나하나씩 대입하는 것이다.  
```
[2]
[2,2]
[2,2,2]
[2,2,2,2]
[2,2,2,3]
[2,2,2,6]
[2,2,2,7]
[2,2,3] 2+2+3 is 7, we don't need go down to [2,2,3,2]
[2,2,6]
[2,2,7]
[2,3]
.
.
.
```
하나하나씩 나열하면 아래와 같이 자식노드가 4개인 트리형식의 표가 만들어진다.
```
                                2  
        22      |       23        |      26         |       27  
222 223 226 227 | 232 233 236 237 | 262 263 266 267 | 272 273 276 277  
.
.
.
                                3  
        32      |       33        |      36         |       37  
322 333 326 327 | 323 333 336 337 | 326 336 366 267 | 327 337 367 377  
.
.
.
```

그런데 이런식으로 하다 보면 [2,2,3][2,3,2][3,2,2] 같은 중복이 발생한다.  
난 여기서 막혀서 거의 1시간정도 끙끙거렸다.  
이것 저것 시도해보다 솔루션을 찾아보니 의외로 간단한 방법으로 해결 할 수 있었다.  
루프를 돌거나 재귀를 할때마다 시작점을 하나씩 올리는 것이다.  
예를 들어 0..Input.len 부터 시작해서 1..Input.len, 2..Input.len 방식으로 시작점을 하나씩 올리면 아래와 같은 순서대로 실행될거다.
```
                                2  
        22      |       23        |      26         |       27  
222 223 226 227 |     233 236 237 |         266 267 |            277  
.
.
.
                                3  
                |       33        |      36         |       37  
                |     333 336 337 |         366 267 |            377  
.
.
.
```
사실 순서에 대한 이해가 있다면 간단한 문제이다.  
앞에서 2부터 시작할때는 **2를 포함한 모든 조합(가능성)** 이 이미 나열된다. 그래서 그 다음에 다시 루프를 시작할때 2를 포함 시켜서 조합을 만들면 안되는 것이다. 이제는 2가 포함된 모든 조합이 나왔으니 2가 포함되지 않는 조합만 나열하면 된다. 그래서 3부터 시작하는 것이고, 마찬가지로 3을 포함한 모든 조합이 완성되고나서는 이제는 3을 배제하고 6으로 시작하는 조합을 나열하면 된다.

### Solution
```rs
pub fn combination_sum(candidates: Vec<i32>, target: i32) -> Vec<Vec<i32>>{
    let mut output: Vec<Vec<i32>> = vec![];
    let mut path = vec![];
    Solution::dfs(0,&mut path, &candidates, target,&mut output);
    output
}
pub fn dfs(i:usize, path:&mut Vec<i32>, can:&[i32], target: i32, output: &mut Vec<Vec<i32>>){
    let sum = path.iter().fold(0,|p,n|p+n);   
    if sum == target{
        output.push(path.clone());
        return;
    }
    if sum>target || i >= can.len(){
        return;
    }
    for j in i..can.len(){  
        path.push(can[j]);
        Solution::dfs(j,&mut path.clone(),can,target,output);
        path.pop();
    }
}
```

## 40. Combination Sum II
### 문제
Given a collection of candidate numbers (candidates) and a target number (target), find all unique combinations in candidates where the candidate numbers sum to target.

Each number in candidates may only be used once in the combination.

Note: The solution set must not contain duplicate combinations.

### Example 1
```
Input: candidates = [10,1,2,7,6,1,5], target = 8
Output: 
[
    [1,1,6],
    [1,2,5],
    [1,7],
    [2,6]
]
```

### Example 2
```
Input: candidates = [2,5,2,1,2], target = 5
Output: 
[
    [1,2,2],
    [5]
]
```
숫자를 중복해서 사용할 수 없다는 점만 빠면 Combinationn Sum I 문제와 같다. 예제에서 [1,1,6]이 가능한 이유는 숫자를 중복해서 사용한게 아니라 주어진 숫자중 1이 두번 있어서 가능한 것 뿐이다.  

DFS 방식으로 모든 노드를 방문하는 방법을 쓰면 앞서 마주했던 중복되는 문제를 다시 마주친다.  
그럼 Combination Sum 1에서 했던 것 처럼 매번 루프를 돌때마다 숫자를 하나씩 배재하는 방법을 쓰면 될까?  
Combination Sum 1에서는 array안에 있는 숫자들이 각각 유니크한 값을 가졌기 때문에 가능했지만 여기서는 중복되는 값이 있기 때문에 그 방법을 그대로 쓰기는 어렵다. 그래서 한가지 스텝을 더해줘야한다.
[10,1,2,7,6,1,5]를 예를 들어보자. 10이 포함된 모든 조합을 만들고 다음에 1이 포함된 조합을 만들려고 하는데 [10,1,2,7,6,1,5]안에는 1이 두번 등장하기 때문에 언젠가는 1이 포함된 조합이 다시 나오고 만다.  
중복을 피하는 방법은 먼저 정렬을 해서 [10,1,2,7,6,1,5] 을 [1,1,2,5,6,7,10] 로 만들고 이미 한번 만들었던 조합이 나오면 다음 숫자로 넘어가는 것이다.  
```
                                1  
        11      |       12        |      15         |       16  
112 115 116 117 |      125 126 127|         156 157 |               167  
.
.
.
```
한번 보자. 첫번째 1로 만들수 있는 조합들의 일부를 나타냈다. 두번째 1을 시작으로 조합을 만들면 112,115,116,117 같이 1이 두번 들어가는 조합은 안나오지만 12,15,16,125,126,127,156,157 같이 1이 한번만 들어가는 조합은 다시 나올 수 있다. 그래서 이미 한번 나온 숫자는 스킵해야하는 것이다. 

### Solution
```rs
pub fn combination_sum2(mut candidates: Vec<i32>, target: i32) -> Vec<Vec<i32>> {
    let mut path = vec![];
    let mut output = vec![];
    candidates.sort();
    Self::dfs(0, path, &candidates, target, &mut output);
    output
}
pub fn dfs(i:usize, mut path: Vec<i32>, can:&[i32],target:i32, output:&mut Vec<Vec<i32>>){
    if target ==0 {
        output.push(path.clone());
        return;
    } else if target < 0{
        return;
    }

    for j in i..can.len(){
        if j > i && can[j] == can[j - 1] {
            continue;
        }
        path.push(can[j]);
        Self::dfs(j+1,path.clone(),can,target-can[j],output);
        path.pop();
    }
}
```
Combination Sum 1과 달라진점은 
1. sort
2. array에서 이미 확인한 숫자는 skip
3. 재귀로 함수 호출할때 인덱스에+1을 더해서 같은 숫자를 숫자 중복쓰기 피하기