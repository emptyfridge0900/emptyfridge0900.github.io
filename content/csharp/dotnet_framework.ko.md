+++
title="CLI? CIL? CLR?"
date=2025-07-09

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++


인터뷰 준비를 하면서 예상 질문들을 찾아보면 꼭 이런 질문들이 나온다.

- .NET이란 무엇인가?
- CIL은 무엇인가?
- CLR은 무엇인가?

그동안 이런 질문들은 대충 넘기고 문법이나 아키텍처 같은 것에만 집중했는데, 오늘은 잠깐 삼천포로 빠져 보았다.

이런 용어들의 기원을 따라가다 보면 끝이 없다. CIL을 보면 bytecode가 나오고, machine code가 나오고, compiled code와 interpreted code가 나오고, JIT, runtime, cross-platform 같은 말이 줄줄이 따라 나온다.

### Common Intermediate Language
가장 낮은 단계의 코드는 machine code다. CPU가 직접 실행할 수 있는 코드다.

machine code는 사람이 읽기 어렵기 때문에, 사람이 조금 더 이해할 수 있는 명령어와 machine code를 대응시킨 것이 assembly라고 볼 수 있다. CPU마다 지원하는 machine code가 다르고, 그에 맞는 assembly도 다르다.

assembly로 직접 개발하기에는 생산성이 낮기 때문에 compiler가 등장했다. C 같은 high-level language로 코드를 작성하면 compiler가 target machine에서 실행 가능한 machine code로 변환해 준다.

compiled language와 interpreted language의 구분은 단순히 "개발이 어렵다/쉽다"로 나누기에는 훨씬 복잡하다. 여기서 중요한 점은 .NET이 source code를 곧바로 특정 CPU용 machine code로만 compile하지 않는다는 것이다.

오늘의 주제인 Intermediate Language, 즉 중간 언어는 말 그대로 중간 단계에 있는 언어다.

C#, F#, VB 같은 .NET 언어를 compile하면 보통 assembly(`.dll`/`.exe`)가 나오고, 그 안에는 metadata와 Intermediate Language가 들어간다. .NET에서는 이 IL을 CIL(Common Intermediate Language)이라고도 부른다.

CIL은 실행 시 CLR이 대부분 Just-In-Time(JIT) compile을 통해 machine code로 바꿔 실행한다. 따라서 ".NET은 compile된 CIL을 그냥 interpret한다"라고만 말하면 정확하지 않다. 현대 .NET에는 JIT뿐 아니라 ReadyToRun, Native AOT 같은 ahead-of-time 옵션도 있다.

.NET에서는 CIL이라고 부르고, Java에서는 비슷한 중간 표현을 보통 bytecode라고 부른다.

### Common Language Runtime
이제 CIL이 무엇인지 대략 알았다. 그렇다면 CLR은 무엇일까?

간단히 말하면 CLR은 .NET runtime environment다. 이미 한 번 compile된 중간 언어(CIL)를 실행 시점에 machine code로 바꾸고, 실행 과정 전체를 관리하는 역할을 한다고 보면 된다.

자세한 내용은 [Just In Time](https://en.wikipedia.org/wiki/Just-in-time_compilation)이나 [Managed Code](https://en.wikipedia.org/wiki/Managed_code)를 읽어보는 것이 좋다.
JIT 은 컴파일 하는 방식이고 Managed Code는 **실행이 CLR에 의해 관리되는 코드** 를 뜻한다.

C#, VB, F# 같은 .NET 언어로 작성한 코드를 컴파일하면 CIL이 나오고, CLR이 그걸 JIT으로 머신코드로 바꿔서 실행한다 — 이 실행 과정 전체를 CLR이 "관리"한다는 의미다.

CLR이 managed code에 제공하는 것들:
- **자동 메모리 관리 (GC)**: 더 이상 사용하지 않는 객체를 자동으로 해제
- **타입 안전성**: 잘못된 캐스팅, 잘못된 메서드 호출, 메모리 크기 오류 방지
- **보안 경계**
- **예외 처리**

C나 C++ 같은 unmanaged 코드는 이것들을 프로그래머가 직접 다 챙겨야 한다. Managed code는 그 부담을 CLR이 대신 져준다.

CIL 자체가 managed code이기도 하지만, managed code는 "중간 언어"라는 형식의 이름이 아니라 "CLR이 관리하는 실행 환경 전체"를 가리키는 개념이다.

### Common Language Infrastructure
그럼 CIL은 compile된 중간 언어이고, CLR은 그 중간 언어를 실행해 주는 runtime이라면, CLI는 무엇일까?

CLI(Common Language Infrastructure)를 직역하면 공통 언어 기반 시설 정도가 되는데, 한국어로는 감이 잘 오지 않는다. 간단히 말하면 기술 명세서다. "runtime, type system, metadata, 중간 언어를 이런 규칙에 맞게 구현하세요"라고 정해 놓은 표준이라고 보면 된다.

CLI의 runtime 영역을 표준 문서에서는 VES(Virtual Execution System)라고 부르고, Microsoft의 구현체가 CLR이다. 현대 .NET에서는 CoreCLR이 그 역할을 한다.

.NET은 runtime만 뜻하지 않는다. runtime, base libraries, SDK, compilers, app stack(ASP.NET Core, Windows Forms, WPF 등)을 포함한 개발 플랫폼이다. CLR/CoreCLR은 그중 실행을 담당하는 핵심 runtime이다.


### 3줄요약
1. 아... 공부하기 싫다
2. 정리해야 할게 너무 많다. 그냥 아래 링크를 따라가 읽자
3. 면접을 10번 넘게 봤지만, 어차피 이런 건 잘 안 물어본다.

### Ref
https://blog.naver.com/ya3344/223125448918
https://en.wikipedia.org/wiki/Common_Intermediate_Language
https://en.wikipedia.org/wiki/Common_Language_Runtime
https://www.sysnet.pe.kr/2/0/1697
https://blog.naver.com/leusin/223697119291
https://namu.wiki/w/%EA%B8%B0%EA%B3%84%EC%96%B4
https://learn.microsoft.com/en-us/dotnet/core/introduction
https://learn.microsoft.com/en-us/dotnet/standard/managed-code
https://learn.microsoft.com/en-us/dotnet/standard/clr
https://learn.microsoft.com/en-us/dotnet/standard/assembly/
https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/introduction
https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/
