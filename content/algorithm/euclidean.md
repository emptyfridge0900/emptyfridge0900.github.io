+++
title="Euclidean Algorithm (유클리드 호제법)"
date=2025-01-09

[taxonomies]
categories = ["Algorithm"]
tags = ["Algorithm","Euclidean"]
+++

Leet code문제를 풀다 Greatest Common Divisor of Strings이라는 문제를 만났다. 처음에는 아무생각 없이 문자열 두개를 포인터를 써서 비교해가는 방법으로 풀었는데 곧 이 방법은 틀렸다는 것을 알았다. 역시 풀기전에 문제를 잘 읽고 분석을...

핵심은 공통된 최대의 문자열을 찾는 것. 최대 공약 문자열 찾는 방법을 문자열의 길이로 바꾸면? 그렇다, 그냥 최대공약수 찾기 문제인것이다.

최대공약수 찾기를 구현해본적이 없어서 혼자 이리저리 생각해 보았는데 결국은 다른 사람의 도움을 받기로 했다. 그리고 정말 깔끔한 코들를 발견했다

```C#
public int gcd(int a, int b) {
    return (b == 0)? a : gcd(b, a % b);
}
```

그리고 이게 그 유명한 유클리드 호제법이라는 것을 알게 되었다.

내가 알던 공약수 구하는 방법은 두 수의 약수들을 구하고 그 약수들을 곱하는 식이였다
예를 들어 252와 105의 공약수를 구하려면 252/3=84, 105/3=35 그리고 다시 84/7=12, 35/7=5, 12와 5는 공약수가 없기 때문에 3과 7이 252와 105의 공약수이고 3*7=21이 최대 공약수인 것이다.




참고 \
<https://www.khanacademy.org/computing/computer-science/cryptography/modarithmetic/a/the-euclidean-algorithm>
<https://seunghyum.github.io/algorithm/Euclidean-algorithm>





