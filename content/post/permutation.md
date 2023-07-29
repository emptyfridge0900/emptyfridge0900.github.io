+++
title="Permutation"
date=2023-05-29

[taxonomies]
categories = ["post"]
tags = ["math"]
+++


몇일전에 leetcode 연습문제 441. Arranging Coins을 풀면서 n개 숫자를 더하는 공식을 보았다.
학교에서 1부터 100까지의 숫자를 더하는 문제를 배워본적이 있지 않은가? 가우스가 10살때 생각해냈다는 그 공식이다.
1부터 n까지의 합은 n(n+1)/2 라는 공식으로 풀수 있다. 풀이는 인터넷에 많이 있으니까 굳이 내가 할 필요는 없는거 갈다.
근데 이 공식을 어디서 많이 본거 같다는 생각이 들었는데, 오늘 갑자기 n(n-1)/2 라는 공식을 배운 기억이 떠올라서 구글링을 해보았다.

## $\frac{n(n-1)}{n}$ n개 중에서 가능한 조합
구글링 해보니 Total number of subarry라고 나온다. 또는 the number of pairs you can form from an *n* element set 이라고 나온다. [링크](https://math.stackexchange.com/questions/2214839/exactly-how-does-the-equation-nn-1-2-determine-the-number-of-pairs-of-a-given)

| | A | B | C | D | E   |
| --- |---|---|---|---|---|
| A |  | AB | AC | AD | AE | 
| B | BA |  | BC | BD | BE |
| C | CA | CB |  | CD | CE |
| D | DA | DB | DC |  | DE |
| E | EA | EB | EC | ED |  |

handshake문제다. n명이 중복되지 않고 악수를 한다고 생각해보자. 
1명이 악수해야 할 숫자는 n명에서 자기 자신을 제외한 n-1번이다.
그럼 n명이 악수해야 할 숫자는 n(n-1)일까?
n(n-1)에는 중복이 포함되있다. AB-BA, AC-CA, AD-DA, AE-EA...





# Permutation
조합문제를 공부하다 보니 수열도 다시 공부하게 되었다.
예전에 문제풀이 하다가 순열에 대해서 배웠는데 지금은 다 까먹었다. 그래서 다시 공부해본다
수열은 조합은 가끔 헷갈린다. 

## $n^r$ n개중에서 r개를 중복가능, 순서를 고려해서 고른다.
ABC가 있는데 그중에서 중복 가능, 순서를 고려해서 3개를 고른다면,

AAA, AAB, AAC
ABA, ABB, ABC 
ACA, ACB, ACC
BAA, BAB, BAC
BBA, BBB, BBC
BCA, BCB, BCC
CAA, CAB, CAC
CBA, CBB, CBC
CCA, CCB, CCC



## $\frac{n!}{(n-r)!}$ n개 중에서 r개를 중복불가, 순서를 고려해서 고른다.
중복불가이기 때문에 한번 고를 때마다 그 전 숫자에서 1을 뺀 만큼의 선택지가 있다.
16개의 당구공 중에서 3개를 선택한다면, 첫번째는 16개 중에서 고르고, 그다음은 15개 중에서 고르고, 그 다음은 14개 중에서 고르게 된다. 16 x 15 x 14 = 3360 의 순열을 얻을 수 있다.
16 x 15 x 14 을 일반화한 공식을 얻기 위해서 factorial을 사용한다.
$\frac{16 * 15 * 14 * 13 * 12 * 11..3 * 2 * 1}{13 * 12 * 11 ... 3 * 2 * 1}$ => $\frac{n!}{(n-r)!}$

## Combination
## $\frac{n!}{r!(n-r)!}$ 조합, 순열에서 순서를 배재한다
ABC나 CBA나 BCA나 A,B,C로 이루어져있기는 매한가지다. 그래서 A,B,C가 들어가면 그냥 ABC 조합이라고 한다.
위에서 $\frac{n!}{(n-r)!}$으로 중복불가하고 순서가 상관있는 순열을 얻었다. 여기서 순서에 상관없이 중복불가한 뽑기를 하면 바로 조합이 된다.
다시 당구공 예제로 돌아가보자. 16 x 15 x 14 = 3360의 순열이있다. 배치 가능한 숫자만큼 나눠주면 조합이 된다.
그렇다면 공3개는 몇가지로 배치가 가능한가? 3! 만큼 가능하다. 공 4개는 4! 만큼 가능하다. 그럼 r개는 r!.
3360순열을 3! 로 나눠주면 560이 나온다.

