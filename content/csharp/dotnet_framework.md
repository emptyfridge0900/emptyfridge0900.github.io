+++
title="CLI? CIL? CLR?"
date=2025-07-09

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++


인터뷰 준비중에 예상 질문들을 찾아보면 꼭 이런 질문들이 있다.  
What is .NET?  
What is CIL?  
What is CLR?  
  
그동안에 이런 질문들은 무시했고 문법이나 아키텍쳐 같은 것에만 집중했는데 오늘은 잠깐 삼천포로 빠져보았다.
  
이런 용어들의 기원을 따라가다보면 정말 끝이 없다. cil을 보면 byte code가 나오고, machine code가 나오고, compile code가 나오고, interprete code가 나오고, Jit이 나오고, runtime이 나오고, cross platform이 나오고....
  
  
### Common Intermediate Language
일단 제일 낮은 단계의 코드는 machine code이다. 컴퓨터가 바로 읽을 수 있는 코드.  
그 machine code는 인간이 읽기 힘드니까 인간이 읽을 수 있는 수준의 코드와 machine code를 매칭시켜주는 것이 assembly라고 들었다.  
모든 cpu는 각자에게 맞는 machine code가 있고 또한 각자에게 맞는 어셈블리코드가 있다.  
어셈블리어로 개발하기에는 생산성이 떨어지니까 컴파일러가 등장했고 C 같은 하이레벨 언어(?)로 코딩하면 이제 컴파일러가 알아서 machine code로 변환해준다.  
컴파일언어가 개발도 어렵고 복잡해서 인터프리터 언어가 탄생했다고 알고 있는데 구체적인 역사적 배경은 모르겠다. 일단 인터프리터 언어는 컴파일 언어와는 다르게 코드를 읽어가며 번역하는 언어이다.  
그리고 오늘의 주제인 Intermdediate Lanugage (중간 언어)가 등장하는데 중간언어는 말 그대로 중간에 있는 언어이다.  
먼저 소스코드를 중간언어,바이트코드 등으로 컴파일하고, 컴파일 된 코드를 인터프리트하는 방식으로 작동한다.  
인터프리트 언어의 읽으면서 번역하는 장정도 챙기고 미리 컴파일 가능한 부분은 컴파일해서 속도를 높였다.  
작동하는 속도도 컴파일언어와 인터프리트 언어 사이이다.
.NET CLR에서는 CIL이라고 부르고 Java에서는 ByteCode라고 부른다.  
  
### Common Language Runtime
일단 CIL이 뭐하는 녀석인지 알겠는데 그럼 CLR은 뭘까?  
간단하게 말하자면 런타임환경이다. 바로 이놈이 이미 한번 컴파일을 거친 중간언어를 머신코드로 바꿔주는 역할을 한다고 생각하면 된다.  
자세한 내용은 [Just In Time](https://en.wikipedia.org/wiki/Just-in-time_compilation) 이나 [Managed Code](https://en.wikipedia.org/wiki/Managed_code) 읽어보는게 좋다.  
JIT 은 컴파일 하는 방식이고 Managed Code는 Intermediate Code을 일컫는 다른 말이다. 모든 중간언어가 managed code는 아니지만 지금은 그렇게 생각하고 넘아가자.  
  

### Common Language Infrastructure
그럼 CIL은 컴파일된 중간언어고 CLR은 중간언어를 실행시켜주는 실행기인데, 그럼 CLI는 뭔가?  
직역하면 공통언어기반시설? 한국어로는 감이 안온다. 간단하게 말하자면 기술 명세서이다. 공통언어기반시설 보다는 좀더 잘 와닿지 않는가?  
CLR은 이러이러한 규칙을 따라서 윈도우에서, 리눅스에서, 맥에서 이렇게저렇게 동작하게 하세요 라고 정해놓은 규칙이라고 생각하면 될거같다.  
그래서 CLI의 요구사항 대로 runtime을 구현한 구현체가 CLR인것이고 더 넓은 의미로는 .NET 그 자체이다.  
.NET은 CLR을 포함하여 앱을 개발할 수 있게하는 요소(라이브러리, 컴파일러 등)를 집어넣은 CLI를 구현한 플랫폼이다.  


### 3줄요약
1. 아... 공부하기 싫다
2. 정리해야 할게 너무 많다. 그냥 아래 링크를 따라가 읽자
3. 면접 10번넘게 봤지만 어차피 이런거 안물어본다. 

### Ref
https://blog.naver.com/ya3344/223125448918  
https://en.wikipedia.org/wiki/Common_Intermediate_Language  
https://en.wikipedia.org/wiki/Common_Language_Runtime  
https://www.sysnet.pe.kr/2/0/1697  
https://blog.naver.com/leusin/223697119291  
https://namu.wiki/w/%EA%B8%B0%EA%B3%84%EC%96%B4  
https://learn.microsoft.com/en-us/dotnet/core/introduction  


