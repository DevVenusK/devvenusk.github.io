---
layout: post
title: Swift Testing
date: "2024-04-30 00:00:00 +0900"
categories: ["iOS/interview"]
tags:
- iOS
- interview
type: post
published: true
meta: {}
---
  
## Test Function
`@Test` Property 를 사용하여 `Test Function` 을 구현한다.
`Test Function` 은 `Global Function` `Type Method` 모두 가능하다.
`async` `throws` `@MainActor` 등이 가능하다
## Expect
`#expect` Macro 를 이용해 `Condition true` 를 확인한다.
`#expect` Macro 는 `Operator`, `Method` 를 포함한 모든 표현식을 전달 가능하다.
```swift
#expect("A" == "B")
#expect(array.isEmpty())
```
## Require
`try #require` Macro 를 사용하면 테스트가 실패시 더이상 진행하지 않는다.
주로 `nil` 을 `binding` 할 때 사용한다.
```swift
let ID = try #require(user.id)
#expect(ID == "0")
```
## Test Suit
`Test Function` 및 `Test Suit` 를 구조화 하여 관리할 수 있다.
`Instance Property` 및 `init`, `deinit` 이 사용가능하다.
`Sturct`, `Class`, `Actor` 로 구현할 수 있으나, `Struct` 를 추천한다.

`Instance Property`는 각 `Test Function` 에 대해 별도로 생성되어 공유상태를 방지한다.
`init`, `deinit` 는 각 `Test Function` 전후에 별도로 호출된다.
```swift
@Suit
struct TestSuit { 
	@Suit
	struct ChildrenTestSuit{ 
		@Test A() { }
		@Test B() { }
	}
}
```
## Test Trait
`endabled`, `disabled`, `tag` 등으로 테스트의 유지보수를 도와줄 수 있는 `Trait` 들이 존재한다.
```swift
@Test(.disbaled)
func A() { }

@Test(.tag(.loans))
func B() { }

@Test(
	.disabled,
	.bug("Known Bug)
	)
func C() { }

@available(*, )
func D() { } // #available 은 사용하지 않는것을 추천한다.
```
## Multi Parameter Test Function
`@Test` Macro 는 `Multi Parameter` 를 지원한다.
```swift
AS-IS
func test_multiParameter() { 
	let alphabet = ["A", "B", "C"]
	for character in alphabet {
		#expact()
	}
}

TO-BE
@Test(arguments: [ 
	"A",
	"B",
	"C"
])
func multiParameter(character: String) async { 
	#expect()
}
```
`Multi Parameter` 는 `async` 하게 실행되며 `for in` 과는 다르게 각 테스트의 실패가 다음 테스트에 영향을 미치지 않는다.
