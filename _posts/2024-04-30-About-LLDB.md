---
layout: post
title: About LLDB
date: "2024-04-30 22:57:00 +0900"
categories: ["iOS/debug"]
tags:
- iOS
- debug
- lldb
type: post
published: true
meta: {}
---
About LLDB
`LLDB`는 `Clang` 컴파일러 인프라를 사용하여 디버그 정보를 `Clang` 타입으로 변환합니다. 이를 통해 최신 `C`, `C++`, `Objective-C`, `Objective-C++` 언어 기능과 Runtime을 지원할 수 있다. 또한, 표현식을 위한 함수 호출 시 모든 `ABI(Application Binary Interface)` 세부사항을 처리하거나, 명령어를 분해하고 명령어 세부사항을 추출하는 등의 작업을 컴파일러가 수행하게 한다.   

- C, C++, Objective-C 언어에 대한 최신 지원
- 로컬 변수와 타입을 선언할 수 있는 다중 행 표현식 지원
- 지원되는 경우 JIT(Just-In-Time 컴파일)을 사용하여 표현식을 실행
- JIT를 사용할 수 없는 경우 표현식의 중간 표현(IR)을 평가

## Command Structure
```
<noun> <verb> [-options [option-value]] [argument [argument...]]
```
각 값들은 공백으로 구별되며 `argument` 공백을 보호하기 위해 작은 따옴표 및 큰 따옴표를 사용할 수 있다. 만약 큰 따옴표나 백슬래시 문자를 `argument` 안에 넣고 싶다면 백슬래시를 사용하여 escape 처리를 하여야 한다.   

`LLDB`에는 특별한 인용 문자가 하나 더 있는데, 그것은 `backtick`(\`)이다. 인자나 옵션 값 주위에 `backtick` 를 사용하면, `LLDB`는 그 값을 표현식 해석기를 통해 실행하고, 그 결과를 명령어에 전달한다.   
```
(lldb) memory read -c `len` 0x12345
```

`LLDB command line`에서 `option`의 위치는 자유롭지만 `option`을 구분하기 위해 `-`를 사용한다.   
```
(lldb) process launch --stop-at-entry -- -program_arg value

(lldb) command run --file -- -mydata.txt

(lldb) breakpoint set --file "example.cpp" --line 50 --condition -- -value == 10
``` 

### 기타
*JIT (Just-In-Time)*   
`JIT` 컴파일러는 프로그램을 실행하는 도중에 필요할 때 코드를 실시간으로 컴파일하는 기술이다.   

*IR (Intermediate Representation)*   
`IR`은 고수준 소스 코드와 낮은 수준의 기계 코드 사이의 중간 단계를 나타낸다. 이는 컴파일러가 소스 코드를 분석하고 최적화하는 과정에서 사용하는, 더 추상적이고 표준화된 코드 형태이다.  

### 참고자료
[LLDB](https://lldb.llvm.org/index.html)    
[LLDB Command map](https://lldb.llvm.org/use/map.html)