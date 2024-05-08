---
layout: post
title: LLDB BreakPoint set Commands
date: "2024-05-09 00:00:00 +0900"
categories: ["iOS/debug"]
tags:
- iOS
- debug
- lldb
- breakpoint
type: post
published: true
meta: {}
---
`XCode`는 `GUI`를 통해 `BreakPoint`를 생성할 수 있다. 하지만 `LLDB Command`를 이용하면 훨씬 다양한 `Option`의 `BreakPoint`를 생성할 수 있다.

```swift
breakpoint set --name viewDidLoad --regex
```   
함수 이름에 `viewDidLoad`를 포함하는 모든 함수에 브레이크포인트를 설정한다.

```swift
breakpoint set --source-regexp 'titleLabel.text = "text"'
```
소스 코드에서 'titleLabel.text = "text"'라는 정확한 문자열을 포함하는 모든 라인에 `BreakPoint`를 설정한다.

```swift
breakpoint set --name 'viewDidLoad' --func-regex
```
함수 이름이 `viewDidLoad`와 정확히 일치하는 함수에 브레이크포인트를 설정한다.

```swift
breakpoint set --file ViewController.swift --line 4
```
`ViewController.swift`파일의 4번째 라인에 브레이크포인트를 설정한다.

```swift
breakpoint set --file ViewController.swift --line 4 --column 10
```
`ViewController.swift` 파일의 4번째 라인, 10번째 칼럼에 `BreakPoint`를 설정한다.

```swift
breakpoint set --name viewDidLoad --shlib MyModule
```
`MyModule` 내의 `viewDidLoad` 함수에 `BreakPoint`를 설정한다. 

```swift
breakpoint set --name viewDidLoad --language swift
```
`Swift`의 `viewDidLoad` 함수에 `BreakPoint`를 설정한다.

```swift
breakpoint set --name 'viewDidLoad' --hardware
```
`viewDidLoad` 함수에 `Hardware BreakPoint`를 설정한다.

```swift
breakpoint set --name exceptionThrow --exception-name NSException --exception-extra-args 'reason'
```
`exceptionThrow`함수에서 `NSException`타입의 예외가 발생할 때, 추가 `argument` `reason`과 함께 `BreakPoint`를 설정한다.

```swift
breakpoint set --address 0x100efd8c0 --offset 5
```
메모리 주소 `0x100efd8c0`에서 `5 bite` `offset`을 둔 위치에 `BreakPoint`를 설정한다.

### 기타
더 다양한 `Option`은 [CommandOption](https://github.com/apple/llvm-project/blob/4898908e9dc0f78befc30bbc307e56a5d8ad20ea/lldb/source/Commands/CommandObjectBreakpoint.cpp#L258) 에서 찾아볼 수 있다.  

### 참고자료
[CommandObjectBreakPoint](https://github.com/apple/llvm-project/blob/next/lldb/source/Commands/CommandObjectBreakpoint.cpp#L258)
