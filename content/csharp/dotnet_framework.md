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
컴파일 언어와 인터프리터 언어의 역사적 구분은 단순히 "개발이 어렵다/쉽다"로 나누기에는 너무 복잡하다. 여기서 중요한 것은 .NET이 소스코드를 바로 CPU용 machine code로만 컴파일하지 않는다는 점이다.  
오늘의 주제인 Intermediate Language(중간 언어)는 말 그대로 중간에 있는 언어이다.  
C#, F#, VB 같은 .NET 언어를 컴파일하면 보통 assembly(.dll/.exe)가 나오고, 그 안에는 metadata와 Intermediate Language가 들어간다. .NET에서는 이 IL을 CIL(Common Intermediate Language)이라고도 부른다.  
이 CIL은 실행 시 CLR이 대부분 Just-In-Time(JIT) 컴파일로 machine code로 바꿔 실행한다. 그러니까 ".NET은 컴파일된 CIL을 그냥 인터프리트한다"라고만 말하면 정확하지 않다. 현대 .NET에는 JIT뿐 아니라 ReadyToRun, Native AOT 같은 ahead-of-time 옵션도 있다.  
.NET에서는 CIL이라고 부르고 Java에서는 보통 bytecode라고 부른다.  
  
### Common Language Runtime
일단 CIL이 뭐하는 녀석인지 알겠는데 그럼 CLR은 뭘까?  
간단하게 말하자면 런타임환경이다. 바로 이놈이 이미 한번 컴파일을 거친 중간언어를 머신코드로 바꿔주는 역할을 한다고 생각하면 된다.  
자세한 내용은 [Just In Time](https://en.wikipedia.org/wiki/Just-in-time_compilation) 이나 [Managed Code](https://en.wikipedia.org/wiki/Managed_code) 읽어보는게 좋다.  
JIT 은 컴파일 하는 방식이고 Managed Code는 **실행이 CLR에 의해 관리되는 코드** 를 뜻한다.

C#, VB, F# 같은 .NET 언어로 작성한 코드를 컴파일하면 CIL이 나오고, CLR이 그걸 JIT으로 머신코드로 바꿔서 실행한다 — 이 실행 과정 전체를 CLR이 "관리"한다는 의미다.

CLR이 managed code에 제공하는 것들:
- **자동 메모리 관리 (GC)**: 더 이상 사용하지 않는 객체를 자동으로 해제
- **타입 안전성**: 잘못된 캐스팅, 잘못된 메서드 호출, 메모리 크기 오류 방지
- **보안 경계**
- **예외 처리**

C나 C++ 같은 unmanaged 코드는 이것들을 프로그래머가 직접 다 챙겨야 한다. Managed code는 그 부담을 CLR이 대신 져준다.

CIL 자체가 managed code이기도 하지만, managed code는 "중간언어"라는 형식의 이름이 아니라 "CLR이 관리하는 실행 환경 전체"를 가리키는 개념이다.  
  

### Common Language Infrastructure
그럼 CIL은 컴파일된 중간언어고 CLR은 중간언어를 실행시켜주는 실행기인데, 그럼 CLI는 뭔가?  
직역하면 공통언어기반시설? 한국어로는 감이 안온다. 간단하게 말하자면 기술 명세서이다. 공통언어기반시설 보다는 좀더 잘 와닿지 않는가?  
CLI는 "이러이러한 규칙을 따라서 runtime, type system, metadata, 중간 언어를 구현하세요"라고 정해놓은 표준이라고 생각하면 된다.  
CLI의 runtime 영역을 표준 문서에서는 VES(Virtual Execution System)라고 부르고, Microsoft의 구현체가 CLR이다. 현대 .NET에서는 CoreCLR이 그 역할을 한다.  
.NET은 runtime만 뜻하지 않는다. runtime, base libraries, SDK, compilers, app stack(ASP.NET Core, Windows Forms, WPF 등)을 포함한 개발 플랫폼이다. CLR/CoreCLR은 그중 실행을 담당하는 핵심 runtime이다.  


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
https://learn.microsoft.com/en-us/dotnet/standard/managed-code  
https://learn.microsoft.com/en-us/dotnet/standard/clr  
https://learn.microsoft.com/en-us/dotnet/standard/assembly/  
https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/introduction  
https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/  
