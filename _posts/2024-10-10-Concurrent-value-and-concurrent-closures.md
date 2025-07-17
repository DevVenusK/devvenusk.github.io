---
layout: post
title: Concurrent-value-and-concurrent-closures
date: 2024-10-10 00:00:00 +0900
categories:
  - swift/evolution/proposals
tags:
  - swift
  - evolution
  - proposals
  - sendable
type: post
published: true
meta: {}
---
- **Proposal**: SE-0302
- **Author**: Chris Lattner
- **Review Manager**: John McCall
- **Status**: Implemented (Swift 5.7)
- **Implementation**: apple/swift#35264

# `Sendable` and `@Sendable` closures
## Introduction
스위프트 동시성 노력의 핵심 목표는 "데이터 경합을 없애기 위해 동시 프로그램에서 상태를 격리하는 메커니즘을 제공하는 것"입니다. 이러한 메커니즘은 널리 사용되는 프로그래밍 언어에서 중요한 진전이 될 것입니다. 대부분의 프로그래밍 언어는 race conditions, deadlcoks 및 기타 문제를 포함한 광범위한 버그에 프로그래머를 노출시키는 방식으로 동시 프로그래밍 추상화를 제공합니다.  

이 제안은 이 분야의 까다로운 문제 중 하나인 structed concurrency constructs 와 actor messages 간에 전달되는 값을 타입 검사하는 방법을 해결하기 위한 접근 방식을 설명합니다. 따라서 이는 두 가지를 안전하고 함께 잘 작동하게 하는 기본 유형 시스템 메커니즘의 일부를 제공하는 통합 이론입니다.  
  
이 구현 방식에는 `Sendable`이라는 marker protocol과 함수에 적용할 수 있는 `@Sendable` attribute가 포함됩니다.
## Motivation  
프로그램의 각 actor 인스턴스와 structed concurrency 작업은 "단일 스레드로 이루어진 섬"을 나타내며, 이는 가변 상태의 가방을 보관하는 자연스러운 동기화 지점이 됩니다. 이들은 다른 작업과 병렬로 연산을 수행하지만, 이러한 시스템의 코드 대부분은 actor의 논리적 독립성을 기반으로 하고 해당 메일박스를 데이터의 동기화 지점으로 사용하여 동기화가 필요하지 않기를 원합니다.  

따라서 핵심적인 질문은 "동시성 도메인 간에 데이터를 언제, 어떻게 전송할 수 있는가?"입니다. 이러한 전송은 예를 들어 structed concurrency에 의해 생성된 actor 메서드 호출 및 작업의 인수와 결과에서 발생합니다.  

Swift Concurrency는 안전하고 강력한 프로그래밍 모델을 구축하고자 합니다. 우리는 다음 세 가지를 달성하고자 합니다.
1. Swift 프로그래머가 보호되지 않은 공유 가변 상태를 유발할 수 있는 동시성 도메인을 전달하려고 할 때 정적 컴파일러 오류가 발생하길 원합니다.  
2. 고급 프로그래머가 다른 사람이 안전하게 사용할 수 있는 정교한 기술(예: a concurrent hash table)이 포함된 라이브러리를 구현할 수 있기를 바랍니다.
3. Swift Concurrency 모델을 염두에 두고 설계되지 않은 코드가 많이 포함된 기존 세계를 포용해야 합니다. 원활하고 점진적인 마이그레이션 스토리가 필요합니다.
  
제안된 솔루션으로 들어가기 전에 모델링할 수 있는 몇 가지 일반적인 사례와 각각의 기회 및 과제를 살펴봅시다. 이를 통해 우리가 다루어야 할 디자인 공간에 대해 추론하는 데 도움이 될 것입니다.  
### 💖 Swift + Value Semantics
우리가 지원해야 할 첫 번째 유형은 정수와 같은 단순한 값입니다. 이러한 값은 pointer를 포함하지 않기 때문에 동시성 도메인에서 간단하게 전달할 수 있습니다.

이보다 더 나아가 Swift는 동시성 경계를 넘어 안전하게 전송할 수 있는 [Value Semantics](https://en.wikipedia.org/wiki/Value_semantics)을 가진 타입에 중점을 두고 있습니다. Swift의 타입 합성 메커니즘은 class가 아닌 경우, 그 구성 요소들이 value Semantics을 제공할 때 value Semantics을 유지합니다. 여기에는 generic structs뿐만 아니라 core collection도 포함됩니다. 예를 들어, `Dictionary<Int, String>`은 동시성 도메인 간에 직접 공유할 수 있습니다. Swift의 Copy on Write 접근 방식은 collection의 표현을 사전 데이터 복사 없이도 전송할 수 있다는 것을 의미하며, 이는 매우 강력한 사실로 실제로 Swift concurrency 모델을 다른 시스템보다 더 효율적으로 만들 수 있을 것이라고 믿습니다.  

그러나 core collection에 일반 class 참조, 변경 가능한 상태를 캡처하는 closure, 기타 non-value 유형이 포함된 경우 동시성 도메인 간에 안전하게 전송할 수 **없습니다**. 안전하게 전송할 수 있는 경우와 그렇지 않은 경우를 구분할 수 있는 방법이 필요합니다.  
### Value Semantic Composition
struct, enum, tuple은 Swift에서 값을 구성하는 기본 모드입니다. 이들은 모두 동시성 도메인 간에 안전하게 전송할 수 있으며, 그 안에 포함된 데이터 자체만 안전하다면 전송할 수 있습니다.  
### Higher Order Functional Programming
함수형 프로그래밍에 뿌리를 둔 Swift와 다른 언어에서는 함수를 다른 함수에 전달하는 [고차 프로그래밍](https://en.wikipedia.org/wiki/Higher-order_function)을 사용하는 것이 일반적입니다. Swift의 함수는 참조 유형이지만, 예를 들어 빈 캡처 목록이 있는 함수와 같이 동시성 도메인을 가로질러 전달해도 완벽하게 안전한 함수가 많습니다.  

함수의 형태로 동시성 도메인 간에 연산 비트를 전송해야 하는 유용한 이유는 많으며, `parallelMap`과 같은 사소한 알고리즘에도 이 기능이 필요합니다. 이는 더 큰 규모에서도 발생합니다. 예를 들어 다음과 같은 actor 예시를 생각해 보세요.
```swift  
actor MyContactList {
  func filteredElements(_ fn: (ContactElement) -> Bool) async -> [ContactElement] { … }
}
```  
그러면 다음과 같이 사용할 수 있습니다.
```swift  
// Closures with no captures are ok!
list = await contactList.filteredElements { $0.firstName != "Max" }

// Capturing a 'searchName' string by value is ok, because strings are
// ok to pass across concurrency domains.
list = await contactList.filteredElements {
  [searchName] in $0.firstName == searchName
}
```  
함수를 동시성 도메인 간에 전달할 수 있도록 하는 것이 중요하다고 생각하지만, 이러한 함수에서 로컬 상태를 _참조_ 로 캡처하는 것을 허용해서는 안 되며, 안전하지 않은 것을 값으로 캡처하는 것도 허용해서는 안 된다는 점도 우려하고 있습니다. 두 가지 모두 memory safety 문제를 야기할 수 있습니다.  
### Immutable Classes
동시 프로그래밍에서 일반적이고 효율적인 설계 패턴 중 하나는 불변 데이터 구조를 구축하는 것입니다. class 내부의 상태가 절대 변하지 않는다면 동시성 도메인 간에 class에 대한 참조를 전송하는 것은 완벽하게 안전합니다. 이 설계 패턴은 매우 효율적이며(ARC 이상의 동기화가 필요하지 않음), [고급 데이터 구조](https://en.wikipedia.org/wiki/Persistent_data_structure)를 구축하는 데 사용할 수 있으며, 순수 함수형 언어 커뮤니티에서 널리 탐구되고 있습니다.  
### Internally Synchronized Reference Types
동시 시스템에서 일반적인 설계 패턴은 클래스가 "thread-safe" API를 제공하는 것입니다. 명시적 동기화(mutexes, atomics 등)로 상태를 보호하는 것입니다. class에 대한 공용 API는 여러 동시성 도메인에서 사용하기에 안전하므로 class에 대한 참조를 직접 안전하게 전송할 수 있습니다.  

actor 인스턴스 자체에 대한 참조가 그 예입니다. actor 내의 변경 가능한 상태는 actor 메일박스에 의해 암시적으로 보호되므로 pointer를 전달하여 동시성 도메인 간에 전달해도 안전합니다.  
### "Transferring" Objects Between Concurrency Domains
동시성 시스템에서 매우 일반적인 패턴은 한 동시성 도메인이 동기화되지 않은 변경 가능한 상태를 포함하는 데이터 구조를 구축한 다음 raw pointer를 전송하여 사용할 다른 동시성 도메인으로 "hand it off"하는 것입니다. 이 방법은 발신자가 구축한 데이터의 사용을 중단하는 경우에만 동기화 없이 올바르게 작동하며, 그 결과 발신자 또는 수신자만 한 번에 변경 가능한 상태에 동적으로 액세스하게 됩니다.
  
이를 달성하는 데는 안전한 방법과 안전하지 않은 방법이 모두 있습니다(예: 마지막에 있는 Alternatives Considered 섹션의 "exotic" 유형 시스템에 대한 토론을 참조하세요.  
### Deep Copying Classes
참조 유형을 전송하는 안전한 방법 중 하나는 데이터 구조의 딥 카피를 만들어 소스 및 대상 동시성 도메인에 각각 변경 가능한 상태의 복사본이 있도록 하는 것입니다. 이 방법은 대규모 구조의 경우 비용이 많이 들 수 있지만 일부 Objective-C 프레임워크에서 일반적으로 사용되었습니다. 일반적인 합의는 이것이 타입 정의에 암시적인 것이 아니라 _명시적_ 이어야 한다는 것입니다.  
### Motivation Conclusion
이것은 패턴의 샘플일 뿐이지만, 보시다시피 다양한 동시 디자인 패턴이 광범위하게 사용되고 있습니다. 가치 유형을 중심으로 설계하고 구조체 사용을 장려하는 Swift의 디자인은 매우 강력하고 유용한 출발점이지만, 특정 도메인에 대해 고성능 API를 표현하고자 하는 커뮤니티뿐만 아니라 하룻밤 사이에 다시 작성되지 않는 레거시 코드를 사용해야 하는 복잡한 사례에 대해서도 추론할 수 있어야 합니다.  

따라서 라이브러리 작성자가 타입의 의도를 표현할 수 있는 접근 방식을 고려하는 것이 중요하고, 앱 프로그래머가 비협조적인 라이브러리를 소급하여 작업할 수 있어야 하며, 전환 과정에 있는 불완전한 세상에 맞서 우리 모두가 "일을 완수"할 수 있도록 안전뿐만 아니라 안전하지 않은 탈출구를 제공하는 것 또한 중요합니다.  
  
마지막으로, 우리의 목표는 (일반적으로 그리고 이 특별한 경우에 있어서) Swift가 건전하고 사용하기 쉬운 고도로 원칙적인 시스템이 되는 것입니다. 20년 후에는 Swift와 그 궁극적인 동시성 모델을 위해 많은 새로운 라이브러리가 구축될 것입니다. 이러한 라이브러리는 value semantic 타입을 중심으로 구축될 것이지만, 전문 프로그래머가 잠금 없는 알고리즘, 불변 타입 사용 또는 도메인에 적합한 다른 디자인 패턴과 같은 최신 기술을 배포할 수 있어야 합니다. 이러한 API의 사용자가 내부적으로 어떻게 구현되는지 신경 쓸 필요가 없어야 합니다.  
## Proposed Solution + Detailed Design
이 제안의 높은 수준의 설계는 `Sendable` marker protocol, 표준 라이브러리 유형에 의한 `Sendable` 채택, 함수에 대한 새로운 `@Sendable` attribute를 중심으로 이루어집니다.  
  
기본 제안 외에도 향후에는 레거시 호환성 사례를 처리하기 위한 adapter 유형 세트와 Objective-C 프레임워크에 대한 일급 지원을 추가하는 것이 합리적일 수 있습니다. 이에 대해서는 다음 섹션에서 설명합니다.  
### Marker Protocol  
이 제안에서는 "Marker" protocol이라는 개념을 도입했는데, 이는 protocol이 일부 의미론적 속성을 가지고 있지만 전적으로 컴파일 타임 개념으로 런타임에는 영향을 미치지 않음을 나타냅니다. Marker protocol에는 다음과 같은 제한이 있습니다:  
* 어떤 종류의 요구 사항도 가질 수 없습니다.  
* Marker protocol이 아닌 프로토콜로부터 상속할 수 없습니다.  
* Marker protocol은 `is` 또는 `as?` 검사에서 유형으로 이름을 지정할 수 없습니다(예: `x as? Sendable`은 오류입니다).  
* Marker protocol은 non-marker protocol에 대한 조건부 protocol 적합성에 대한 일반 제약 조건에 사용할 수 없습니다.  
  
일반적으로 유용한 기능이라고 생각하지만, 현재로서는 컴파일러 내부 기능이어야 한다고 생각합니다. 따라서 아래 "`@_marker`" 속성 구문으로 이 개념을 설명하고 사용합니다.  
### `Sendable` Protocol
이 제안의 핵심은 Swift 표준 라이브러리에 정의된 marker protocol로, 특별한 적합성 검사 규칙이 있습니다.
```swift  
@_marker
protocol Sendable {}
```  
모든 공용 API가 동시성 도메인에서 안전하게 사용할 수 있도록 설계된 타입은 `Sendable` protocol을 준수하는 것이 좋습니다. 예를 들어 public mutator가 없는 경우, public mutator가 COW로 구현된 경우, 또는 internal locking이나 다른 메커니즘으로 구현된 경우 등이 이에 해당합니다. 물론 public API의 일부로 잠금이나 COW가 있는 경우, 타입은 로컬 돌연변이에 기반한 내부 구현 세부 사항을 가질 수 있습니다.  
  
컴파일러는 actor 메시지 전송 또는 structured concurrency 호출의 인수나 결과가 `Sendable` protocol을 따르지 않는 경우와 같이 동시성 도메인 간에 데이터를 전달하려는 모든 시도를 거부합니다:  
```swift  
actor SomeActor {
  // async functions are usable *within* the actor, so this
  // is ok to declare.
  func doThing(string: NSMutableString) async {...}
}

// ... but they cannot be called by other code not protected
// by the actor's mailbox:
func f(a: SomeActor, myString: NSMutableString) async {
  // error: 'NSMutableString' may not be passed across actors;
  //        it does not conform to 'Sendable'
  await a.doThing(string: myString)
}
```  
`Sendable` protocol은 값을 복사하여 동시성 도메인 간에 안전하게 전달할 수 있는 타입을 모델링합니다. 여기에는 값 의미형, 불변 참조형에 대한 참조, 내부적으로 동기화된 참조형, `@Sendable` closure, 고유 소유권 등을 위한 향후 다른 타입 시스템 확장이 포함됩니다.

이 protocol을 잘못 준수하면 프로그램에 버그가 발생할 수 있으므로(`Hashable`을 잘못 구현하면 불변성이 깨질 수 있는 것처럼) 컴파일러가 준수 여부를 검사합니다(아래 참조).    
#### Tuple conformance to `Sendable`
Swift에는 특정 protocl에 대한 [튜플에 대한 하드 코딩된 적합성](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0283-tuples-are-equatable-comparable-hashable.md)이 있으며, 튜플 요소가 모두 `Sendable`을 준수하는 경우 이를 `Sendable`로 확장해야 합니다.  
#### Metatype conformance to `Sendable`
메타타입(예: `Int.Type`, `Int.self` 표현식에 의해 생성되는 타입)은 불변이므로 항상 `Sendable`을 준수합니다.  
#### `Sendable` conformance checking for structs and enums
`Sendable` 유형은 Swift에서 매우 일반적이며 이러한 유형의 집합은 동시성 도메인 간에 전송하는 데에도 안전합니다. 따라서 Swift 컴파일러는 다른 `Sendable` 타입의 구성인 struct와 enum에 대해 `Sendable`을 직접 준수할 수 있습니다.
```swift
struct MyPerson : Sendable { var name: String, age: Int }
struct MyNSPerson { var name: NSMutableString, age: Int }

actor SomeActor {
  // Structs and tuples are ok to send and receive!
  public func doThing(x: MyPerson, y: (Int, Float)) async {..}

  // error if called across actor boundaries: MyNSPerson doesn't conform to Sendable!
  public func doThing(x: MyNSPerson) async {..}
}
```  
이 방법은 편리하지만, 더 많은 고려가 필요한 경우 protocol 채택의 마찰을 약간 증가시키고자 합니다. 따라서 컴파일러는 struct와 enum의 멤버(또는 연관된 값) 중 하나가 `Sendable`을 따르지 않거나 일반 제약 조건을 통해 `Sendable`을 따르는 것으로 알려지지 않은 경우 `Sendable` protocol에 대한 준수를 거부합니다.
```swift  
// error: MyNSPerson cannot conform to Sendable due to NSMutableString member.
// note: add '@unchecked' if you know what you're doing.
struct MyNSPerson : Sendable {
  var name: NSMutableString
  var age: Int
}

// error: MyPair cannot conform to Sendable due to 'T' member which may not itself be a Sendable
// note: see below for use of conditional conformance to model this
struct MyPair<T> : Sendable {
  var a, b: T
}

// use conditional conformance to model generic types
struct MyCorrectPair<T> {
  var a, b: T
}

extension MyCorrectPair: Sendable where T: Sendable { }
```  
컴파일러 진단에서 언급했듯이, 모든 타입은 `@unchecked`로 `Sendable`에 대한 적합성을 주석으로 추가하여 이 검사 동작을 재정의할 수 있습니다. 이는 해당 타입이 동시성 도메인 간에 안전하게 전달될 수 있음을 나타내지만, 해당 타입의 작성자가 이를 보장해야 합니다.  
  
`struct` 또는 `enum`은 해당 유형이 정의된 동일한 소스 파일 내에서만 `Sendable`을 따르도록 만들 수 있습니다. 이렇게 하면 구조체에 저장된 프로퍼티와 열거형에 연결된 값을 볼 수 있으므로 해당 유형이 `Sendable`을 준수하는지 확인할 수 있습니다.  
```swift  
// MySneakyNSPerson.swift
struct MySneakyNSPerson {
  private var name: NSMutableString
  public var age: Int
}

// in another source file or module...
// error: cannot declare conformance to Sendable outside of
// the source file defined MySneakyNSPerson
extension MySneakyNSPerson: Sendable { }
```  
이 제한이 없으면 비공개 저장 프로퍼티 이름을 볼 수 없는 다른 소스 파일이나 모듈은 `MySneakyNSPerson`이 제대로 `Sendable`이라고 결론을 내릴 것입니다. 이 검사도 비활성화하려면 `Sendable`에 대한 적합성을 `@unchecked`로 선언하면 됩니다.
```swift  
// in another source file or module...
// okay: unchecked conformances in a different source file are permitted
extension MySneakyNSPerson: @unchecked Sendable { }
```  
#### Implicit struct/enum conformance to `Sendable`
많은 구조체와 열거형은 `Sendable`의 요구 사항을 충족하며, 모든 유형에 명시적으로 "`: Sendable`"을 써야 합니다. `Sendable`을 모든 유형에 명시적으로 작성하는 것은 상용구처럼 느껴질 수 있습니다. 비공개 구조체와 열거형도 `@usableFromInline`이 아닌 경우, 그리고 고정된 공개 구조체와 열거형의 경우 (이전 섹션에서 설명한) 적합성 검사에 성공하면 `Sendable` 적합성이 암시적으로 제공됩니다.
```swift  
struct MyPerson2 { // Implicitly conforms to Sendable!
  var name: String, age: Int
}

class NotConcurrent { } // Does not conform to Sendable

struct MyPerson3 { // Does not conform to Sendable because nc is of non-Sendable type
  var nc: NotConcurrent
} 
```  
non-frozen public struct와 enum은 암시적 적합성을 얻지 않습니다. 그렇게 하면 API 복원력에 문제가 생길 수 있기 때문입니다. `Sendable`에 대한 암시적 적합성은 의도하지 않았더라도 API 클라이언트와의 계약의 일부가 될 수 있기 때문입니다. 또한 이 계약은 `Sendable`을 준수하지 않는 저장소로 struct나 enum을 확장함으로써 쉽게 깨질 수 있습니다.  
> **Rationale**: `Hasable`, `Equatable`, `Codable`의 기존 선례는 디테일이 합성된 경우에도 명시적인 준수를 요구하고 있습니다. `Sendable`에 대해서는 이러한 선례를 따르지 않는 이유는 (1) `Sendable`이 훨씬 더 일반화될 가능성이 높고, (2) 다른 프로토콜과 달리 `Sendable`은 코드 크기(또는 바이너리에 전혀 영향이 없으며, (3) `Sendable`이 동시성 도메인에서 유형을 사용할 수 있도록 하는 것 외에 추가 API를 도입하지 않기 때문입니다.  

`Sendable`에 대한 암시적 준수는 generic이 아닌 유형과 인스턴스 데이터가 `Sendable` 유형으로 보장되는 generic 유형에 대해서만 사용할 수 있다는 점에 유의하세요.
```swift  
struct X<T: Sendable> {  // implicitly conforms to Sendable
  var value: T
}

struct Y<T> {    // does not implicitly conform to Sendable because T does not conform to Sendable
  var value: T
}
```  
Swift는 암시적으로 조건부 적합성을 도입하지 않습니다. 향후 제안에서 이 기능이 도입될 가능성이 있습니다.  
#### `Sendable` conformance checking for classes
모든 class는 `@unchecked` 적합성으로 `Sendable`을 준수하도록 선언하여 의미 검사 없이 actor 간에 전달할 수 있습니다. 이는 access control 및 내부 동기화를 사용하여 memory safety를 제공하는 class에 적합하며, 이러한 메커니즘은 일반적으로 컴파일러에서 검사할 수 없습니다.  

또한 class가 `Sendable`을 준수하고 컴파일러가 memory safety를 검사할 수 있는 특정 제한된 경우, 즉 class가 Sendable을 준수하는 타입의 불변 저장 프로퍼티만 포함하는 final class인 경우에만 컴파일러가 메모리 안전성을 검사할 수 있습니다.
```swift
final class MyClass : Sendable {
  let state: String
}
```  
이러한 class는 (Objective-C 상호 운용성을 위해) NSObject 이외의 class에서 상속할 수 없습니다. `Sendable` class는 struct 및 enum과 동일한 제한이 있어 동일한 소스 파일에서 `Sendable` 적합성이 발생해야 합니다.  

이 동작을 통해 actor 간에 공유 상태의 불변 백을 안전하게 생성하고 전달할 수 있습니다. 향후 이를 일반화할 수 있는 몇 가지 방법이 있지만, 명확하지 않은 경우가 있습니다. 따라서 이 제안은 동시성 설계의 다른 측면에 진전을 이루기 위해 의도적으로 class에 대한 안전 검사를 제한적으로 유지합니다.  
#### Actor types
actor 타입은 자체적인 내부 동기화를 제공하므로 암시적으로 `Sendable`을 준수합니다. [actor proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)에서 더 자세한 내용을 확인할 수 있습니다.  
#### Key path literals
키 경로 자체는 `Sendable` protocol을 준수합니다. 그러나 키 경로를 안전하게 공유하기 위해 키 경로 리터럴은 `Sendable` protocol을 준수하는 타입의 값만 캡처할 수 있습니다. 이는 키 경로의 아래 첨자 사용에 영향을 줍니다.
```swift  
class SomeClass: Hashable {
  var value: Int
}

class SomeContainer {
  var dict: [SomeClass : String]
}

let sc = SomeClass(...)

// error: capture of 'sc' in key path requires 'SomeClass' to conform
// to 'Sendable'
let keyPath = \SomeContainer.dict[sc]
```  
### New `@Sendable` attribute for functions
`Sendable` protocol은 값 유형을 직접 다루고 class가 동시성 시스템에 참여할 수 있도록 허용하지만, 함수 유형은 현재 protocol을 준수할 수 없는 중요한 참조 유형이기도 합니다. Swift에서 함수는 전역 함수 선언, 중첩 함수, 접근자(getter, setter, subscripts 등), closure 등 여러 형태로 존재합니다. 예를 들어, `parallelMap` 및 기타 명백한 동시성 구조체의 정의를 허용하는 등, Swift 동시성 모델에서 고차 함수형 프로그래밍 기법을 허용하기 위해 가능한 경우 함수가 동시성 도메인 간에 전달될 수 있도록 하는 것이 유용하고 중요합니다.  
  
우리는 `@Sendable`이라는 함수 유형에 새로운 attribute를 정의할 것을 제안합니다. `Sendable` 함수형은 동시성 도메인 간에 전송하기에 안전합니다(따라서 `Sendable` protocol을 암시적으로 준수합니다). memory safety을 보장하기 위해 컴파일러는 `@Sendable` 함수 형을 가진 값(예: closure 및 함수)에 대해 몇 가지 사항을 검사합니다.
1. 함수는 `@Sendable`로 표시할 수 있습니다. 모든 캡처도 `Sendable`을 준수해야 합니다.  
2. 함수 유형이 `@Sendable`인 closure는 값으로만 캡처를 사용할 수 있습니다. `let`에 의해 도입된 불변 값의 캡처는 암시적으로 값에 의한 캡처이며, 다른 모든 캡처는 캡처 목록을 통해 지정해야 합니다.
```swift  
actor MyContactList {
  func filteredElements(_ fn: @Sendable (ContactElement) -> Bool) async -> [ContactElement] { … }
}
```  
   캡처된 모든 값의 유형은 `Sendable`을 준수해야 합니다.  
1. 이 제안으로 현재 접근자는 `@Sendable` 시스템에 참여할 수 없습니다. 이에 대한 요구가 있다면 향후 제안에서 getter가 참여할 수 있도록 허용하는 것은 간단할 것입니다.  
  
함수 타입에 대한 `@Sendable` 속성은 기존의 `@이스케이프` 속성과 직교하지만 작동 방식은 동일합니다. `Sendable` 함수는 항상 `@Sendable` 함수가 아닌 함수의 하위 유형이며, 필요할 때 암시적으로 변환됩니다. 마찬가지로 closure 표현식도 `@escaping` closure와 마찬가지로 컨텍스트에서 `@Sendable` 비트를 추론합니다.  

동기 부여 섹션의 예제를 다시 살펴보자면 다음과 같이 선언할 수 있습니다:.
```swift  
actor MyContactList {
  func filteredElements(_ fn: @Sendable (ContactElement) -> Bool) async -> [ContactElement] { … }
}
```  
그러면 다음과 같이 사용할 수 있습니다.
```swift  
// Closures with no captures are ok!
list = await contactList.filteredElements { $0.firstName != "Max" }

// Capturing a 'searchName' string is ok, because String conforms
// to Sendable.  searchName is captured by value implicitly.
list = await contactList.filteredElements { $0.firstName == searchName }

// @Sendable is part of the type, so passing a compatible
// function declaration works as well.
list = await contactList.filteredElements(dynamicPredicate)

// Error: cannot capture NSMutableString in a @Sendable closure!
list = await contactList.filteredElements {
  $0.firstName == nsMutableName
}

// Error: someLocalInt cannot be captured by reference in a
// @Sendable closure!
var someLocalInt = 1
list = await contactList.filteredElements {
  someLocalInt += 1
  return $0.firstName == searchName
}
```  
`Sendable` closure와 `Sendable` 타입의 조합은 라이브러리 확장이 가능하면서도 사용과 이해가 쉬운 타입 안전 동시성을 가능하게 합니다. 이 두 가지 개념은 actor와 structured concurrency이 그 위에 구축되는 핵심 기반입니다.  
#### Inference of `@Sendable` for Closure Expressions
closure 표현식에 대한 `@Sendable` 속성에 대한 추론 규칙은 closure `@escaping` 추론과 유사합니다. closure 표현식은 `@Sendable`로 추론됩니다:  
* `@Sendable` 함수 유형을 기대하는 컨텍스트에서 사용되거나(예: `parallelMap` 또는 `Task.runDetached`), 또는  
* closure의 `in` 명세에 `@Sendable`이 있는 경우.  
`@escaping`과 다른 점은 컨텍스트 없는 closure는 기본적으로 `@Sendable`이 아니지만 `@escaping`으로 기본 설정된다는 점입니다.
```swift  
// defaults to @escaping but not @Sendable
let fn = { (x: Int, y: Int) -> Int in x+y }
```  
중첩 함수도 closure 표현식처럼 값을 캡처할 수 있기 때문에 중요한 고려 사항입니다. 중첩 함수 선언에 `@Sendable` 속성을 사용하여 동시성 검사를 선택할 수 있습니다.
```swift  
func globalFunction(arr: [Int]) {
  var state = 42

  // Error, 'state' is captured immutably because closure is @Sendable.
  arr.parallelForEach { state += $0 }

  // Ok, function captures 'state' by reference.
  func mutateLocalState1(value: Int) {
    state += value
  }

  // Error: non-@Sendable function isn't convertible to @Sendable function type.
  arr.parallelForEach(mutateLocalState1)

  @Sendable
  func mutateLocalState2(value: Int) {
    // Error: 'state' is captured as a let because of @Sendable
    state += value
  }

  // Ok, mutateLocalState2 is @Sendable.
  arr.parallelForEach(mutateLocalState2)
}
```  
이렇게 하면 structured concurrency과 actor 모두에 대해 깔끔하게 구성됩니다.  
### Thrown errors
`throw`하는 함수나 클로저는 `Error` protocol을 준수하는 모든 타입의 값을 효과적으로 반환할 수 있습니다. 함수가 다른 동시성 도메인에서 호출되는 경우, 던져진 값이 그 도메인을 가로질러 전달될 수 있습니다.
```swift
class MutableStorage {
  var counter: Int
}
struct ProblematicError: Error {
  var storage: MutableStorage
}

actor MyActor {
  var storage: MutableStorage
  func doSomethingRisky() throws -> String {
    throw ProblematicError(storage: storage)
  }
}
```  
다른 동시성 도메인에서 `myActor.doSomethingRisky()`를 호출하면 문제가 되는 에러가 발생하여 `myActor`의 변경 가능한 상태의 일부를 캡처한 다음 다른 동시성 도메인에 제공하여 actor 격리를 깨뜨리게 됩니다. `doSomethingRisky()`의 시그니처에는 던져지는 에러 유형에 대한 정보가 없고, `doSomethingRisky()`에서 전파되는 에러는 함수가 호출하는 _모든_ 코드에서 발생할 수 있으므로 `Sendable`에 부합하는 에러만 던지는지 확인할 수 있는 곳이 없습니다.  
  
이 안전 구멍을 막기 위해, 우리는 `Error` protocol의 정의를 변경하여 모든 오류 유형이 `Sendable`을 준수하도록 요구합니다.
```swift  
protocol Error: Sendable { … }
```  
이제 `ProblematicError` 유형은 `Sendable`을 준수하지만 `Sendable` 유형이 아닌 `MutableStorage`의 저장 프로퍼티를 포함하고 있기 때문에 오류와 함께 거부됩니다.  
  
일반적으로 소스 및 바이너리 호환성을 모두 깨지 않고는 기존 protocol에 새로운 상속 protocol을 추가할 수 없습니다. 그러나 marker protocol은 ABI에 영향을 미치지 않고 요구 사항도 없으므로 바이너리 호환성이 유지됩니다.  
  
그러나 소스 호환성에는 더 많은 주의가 필요합니다. 현재 Swift에서는 `ProblematicError`가 잘 형성되어 있지만, `Sendable`이 도입되면 거부될 것입니다. 전환을 용이하게 하기 위해 `Error`를 통해 `Sendable` 적합성을 얻는 유형에 대한 오류는 Swift 6 이하에서 에서 경고로 다운그레이드됩니다.
### Adoption of `Sendable` by Standard Library Types
표준 라이브러리 유형이 동시성 도메인 간에 전달되는 것이 중요합니다. 대부분의 표준 라이브러리 유형은 값 시맨틱을 제공하므로 `Sendable`을 준수해야 합니다.
```swift  
extension Int: Sendable {}
extension String: Sendable {}
```  
일반 값 의미형 타입은 모든 엘리먼트 타입이 동시성 도메인 간에 전달되어도 안전하다면 동시성 도메인 간에 전달되어도 안전합니다. 이 종속성은 조건부 적합성으로 모델링할 수 있습니다:  
```swift  
extension Optional: Sendable where Wrapped: Sendable {}
extension Array: Sendable where Element: Sendable {}
extension Dictionary: Sendable
    where Key: Sendable, Value: Sendable {}
```  
아래 나열된 경우를 제외하고 표준 라이브러리의 모든 구조체, 열거형 및 클래스 형은 `Sendable` protocol을 준수합니다. generic type은 모든 generic 인자가 `Sendable`을 준수할 때 조건부로 `Sendable` protocol을 준수합니다. 이러한 규칙의 예외는 다음과 같습니다.
* `ManagedBuffer`: 이 클래스는 버퍼에 대한 변경 가능한 참조 시맨틱을 제공하기 위한 것입니다. 안전하지 않더라도 `Sendable`을 따르지 않아야 합니다.  
* `Unsafe(Mutable)(Buffer)Pointer`: 이러한 제네릭 형은 `Sendable` protocol을 _무조건적으로_ 준수합니다. 즉, 비동시 값에 대한 안전하지 않은 pointer는 잠재적으로 동시성 도메인 간에 해당 값을 공유하는 데 사용될 수 있습니다. 안전하지 않은 pointer 유형은 근본적으로 안전하지 않은 메모리 액세스를 제공하며, 프로그래머가 이를 올바르게 사용하도록 신뢰해야 합니다. 완전히 안전하지 않은 한 가지 좁은 차원의 사용에 대해 엄격한 안전 규칙을 적용하는 것은 이러한 설계에 부합하지 않는 것처럼 보입니다.  
* 지연 알고리즘 어댑터 유형: 지연 알고리즘이 반환하는 유형(예: `array.lazy.map` { ... }의 결과)은 `Sendable`에 절대 부합하지 않습니다. 이러한 알고리즘 중 상당수(예: lazy `map`)는 `@Sendable`이 아닌 closure저 값을 취하므로 `Sendable`을 안전하게 준수할 수 없습니다.
  
표준 라이브러리 protocol인 `Error`와 `CodingKey`는 `Sendable` protocol을 상속합니다:  
* 이전 섹션에서 설명한 대로 `Error`는 `Sendable`에서 상속되어 던져진 에러가 동시성 도메인에 안전하게 전달될 수 있도록 합니다.  
* `CodingKey`는 `Sendable`에서 상속되므로 `CodingKey` 인스턴스를 저장하는 `EncodingError` 및 `DecodingError`와 같은 유형이 `Sendable`에 올바르게 부합할 수 있습니다.  
### Support for Imported C / Objective-C APIs
C 및 Objective-C와의 상호운용성은 Swift에서 중요한 부분입니다. C 코드는 항상 암묵적으로 동시성에 대해 안전하지 않은데, 이는 Swift가 C API의 올바른 동작을 강제할 수 없기 때문입니다. 그러나 많은 C 유형에 대해 암시적인 `Sendable` 준수를 제공함으로써 동시성 모델과의 기본적인 상호 작용을 정의하고 있습니다:  
* C 열거형 타입은 항상 `Sendable` protocol을 준수합니다.  
* C 구조체 유형은 저장된 모든 프로퍼티가 `Sendable`을 준수하는 경우 `Sendable` protocol을 준수합니다.  
* C 함수 pointer는 `Sendable` protocol을 준수합니다. 이는 값을 캡처할 수 없으므로 안전합니다.  
## Future Work / Follow-on Projects
기본 제안 외에도 후속 제안으로 살펴볼 수 있는 몇 가지 후속 작업이 있습니다.  
### Adaptor Types for Legacy Codebases
**NOTE**: 이 섹션은 제안의 일부로 간주되지 않으며 설계의 측면을 설명하기 위해 포함되었습니다.  

위의 제안은 동시성을 지원하도록 업데이트된 컴포지션 및 Swift 타입에 대한 좋은 지원을 제공합니다. 또한 protocol의 소급 준수를 지원하는 Swift는 사용자가 아직 업데이트되지 않은 코드베이스로 작업할 수 있게 해줍니다.  
  
그러나 기존 프레임워크와의 호환성 측면에서 직면해야 할 또 다른 중요한 측면이 있습니다. 프레임워크는 때때로 애드혹 구조를 가진 변경 가능한 객체의 밀집된 그래프를 중심으로 설계됩니다. 언젠가는 "세상을 다시 쓰는" 것이 좋겠지만, 실제 Swift 프로그래머는 그 동안 "일을 처리"하기 위해 지원이 필요할 것입니다. 비유하자면, Swift가 처음 나왔을 때 대부분의 Objective-C 프레임워크는 무효화 가능성에 대한 감사를 받지 않았습니다. 저희는 과도기를 처리하기 위해 "`ImplicitlyUnwrappedOptional`"을 도입했고, 이는 수년에 걸쳐 서서히 사용되지 않게 되었습니다.  

Swift concurrency로 이 작업을 수행하는 방법을 설명하기 위해 Objective-C 프레임워크에서 흔히 볼 수 있는 패턴인 스레드 간에 참조를 "transferring"하여 객체 그래프를 전달하는 것을 생각해 보세요. 이는 유용하지만 메모리 안전하지는 않습니다! 프로그래머는 앱 내에서 이러한 것들을 액터 API의 일부로 표현할 수 있기를 원할 것입니다.  
  
이는 일반 헬퍼 구조체를 도입하면 가능합니다.
```swift  
@propertyWrapper
struct UnsafeTransfer<Wrapped> : @unchecked Sendable {
  var wrappedValue: Wrapped
  init(wrappedValue: Wrapped) {
    self.wrappedValue = wrappedValue
  }
}
```  
예를 들어, `NSMutableDictionary`는 동시성 도메인을 가로지르는 것이 안전하지 않으므로 `Sendable`을 따르는 것이 안전하지 않습니다. 위의 구조체를 사용하면 (앱 프로그래머로서) 애플리케이션에서 액터 API를 다음과 같이 작성할 수 있습니다.
```swift  
actor MyAppActor {
  // The caller *promises* that it won't use the transferred object.
  public func doStuff(dict: UnsafeTransfer<NSMutableDictionary>) async
}
```  
이 방법은 특별히 예쁘지는 않지만, 감사되지 않고 안전하지 않은 코드로 작업해야 할 때 호출자 측에서 작업을 완료하는 데 효과적입니다. 또한 최근에 제안된 [인수를 위한 속성 래퍼로의 확장](https://forums.swift.org/t/pitch-2-extend-property-wrappers-to-function-and-closure-parameters/40959)을 사용하여 매개변수 속성에 설탕을 넣을 수 있으므로 더 예쁜 선언과 호출자 측 구문을 사용할 수 있습니다.
```swift  
actor MyAppActor {
  // The caller *promises* that it won't use the transferred object.
  public func doStuff(@UnsafeTransfer dict: NSMutableDictionary) async
}
```  
### Objective-C Framework Support
**NOTE**: 이 섹션은 제안의 일부로 간주되지 않으며, 단지 설계의 측면을 설명하기 위해 포함되었습니다.  

Objective-C에는 이 프레임워크에 일괄적으로 도입하는 것이 합당한 패턴이 확립되어 있습니다. 예를 들어, [`NSCopying` 프로토콜](https://developer.apple.com/documentation/foundation/nscopying)은 중요하고 널리 채택된 프로토콜 중 하나로 이 프레임워크에 온보딩되어야 합니다.  
  
일반적인 합의는 모델에서 복사를 명시적으로 만드는 것이 중요하다는 것이므로 다음과 같이 `NSCopied` 헬퍼를 구현할 수 있습니다.
```swift  
@propertyWrapper
struct NSCopied<Wrapped: NSCopying>: @unchecked Sendable {
  let wrappedValue: Wrapped

  init(wrappedValue: Wrapped) {
    self.wrappedValue = wrappedValue.copy() as! Wrapped
  }
}
```  
이렇게 하면 액터 메서드의 개별 인자와 결과를 이렇게 복사할 수 있습니다.
```swift  
actor MyAppActor {
  // The string is implicitly copied each time you invoke this.
  public func lookup(@NSCopied name: NSString) -> Int async
}
```  
한 가지 주의할 점은 정적으로 타입이 지정된 `NSString`은 서브클래스 관계로 인해 실제로는 동적으로 `NSMutableString`이 될 수 있기 때문에, Objective-C 정적 타입 시스템은 여기서 불변성과 관련하여 그다지 도움이 되지 않는다는 것입니다. 따라서 `NSString` 타입의 값이 동적으로 불변이라고 가정하는 것은 안전하지 않으며, `copy()` 메서드를 호출하도록 구현해야 합니다.  
### Interaction of Actor self and `@Sendable` closures
actor는 이 제안 위에 개념적으로 겹쳐진 제안이지만, 이 제안이 요구 사항을 충족하는지 확인하기 위해 액터 설계에 유의하는 것이 중요합니다. 위에서 설명한 것처럼 액터 메서드가 동시성 경계를 가로질러 전송하려면 인자와 결과가 `Sendable`을 준수해야 하며, 따라서 이러한 경계를 통과하는 closure는 `@Sendable`이어야 합니다.  
  
추가로 해결해야 할 한 가지 세부 사항은 "언제 교차 actor 호출인가?"입니다. 예를 들어, 우리는 이러한 호출이 동기식이어야 하고 대기가 필요하지 않기를 원합니다.
```swift  
extension SomeActor {
  public func oneSyncFunction(x: Int) {... }
  public func otherSyncFunction() {
    // No await needed: stays in concurrency domain of self actor.
    self.oneSyncFunction(x: 42)
    oneSyncFunction(x: 7)    // Implicit self is fine.
  }
}
```  
하지만 `self`가 액터 메서드 내에서 closure로 캡처되는 경우도 고려해야 합니다.
```swift  
extension SomeActor {
  public func thing(arr: [Int]) {
    // This should obviously be allowed!
    arr.forEach { self.oneSyncFunction(x: $0) }

    // Error: await required because it hops concurrency domains.
    arr.parallelMap { self.oneSyncFunction(x: $0) }

    // Is this ok?
    someHigherOrderFunction {
      self.oneSyncFunction(x: 7)  // ok or not?
    }
  }
}
```  
컴파일러는 동시성 도메인 홉이 가능한지 아닌지를 알아야 하며, 가능하다면 await를 해야 합니다. 다행히도 위의 기본 타입 시스템 규칙을 간단하게 구성하면 이 문제는 해결됩니다. actor 메서드에서 `@Sendable`이 아닌 closure에서 actor `self`를 사용하는 것은 완벽하게 안전하지만, `@Sendable` closure에서 사용하면 다른 동시성 도메인에서 온 것으로 취급되므로 `await`이 필요합니다.
### Marker protocols as custom attributes
marker protocol `Sendable`과 함수 attribute `@Sendable`은 의도적으로 같은 이름을 부여했습니다. 이 제안에서처럼 컴파일러가 인식하는 특수 attribute에서 `@Sendable`과 같은 marker protocol을 [property wrapper](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md) 및 [result builder](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md)와 같은 사용자 정의 attribute로 전환할 수 있는 잠재적인 미래 방향이 여기에 있습니다. 이러한 변경은 사용자가 표준 라이브러리의 것을 가리는 자체 `Sendable` 유형을 선언하지 않는 한 `@Sendable`을 사용하는 기존 코드에는 거의 영향을 미치지 않습니다. 그러나 `@Sendable`의 특수성이 떨어지고 다른 marker protocol을 유사하게 사용할 수 있게 됩니다.  
## Source Compatibility
이것은 기존 코드 베이스와 거의 완벽하게 소스 호환이 가능합니다. `Sendable` marker protocol과 `@Sendable` 함수의 도입은 사용하지 않을 때 아무런 영향을 미치지 않는 추가 기능이므로 기존 코드에 영향을 미치지 않습니다.  
  
예외적인 경우 소스가 손상될 수 있는 몇 가지 새로운 제한 사항이 있습니다:  
* 키 경로 리터럴 첨자로 변경하면 비표준 유형으로 인덱싱된 이국적인 키 경로가 깨집니다.  
* `Error` 및 `CodingKey`는 `Sendable`에서 상속되므로 사용자 정의 오류 및 키가 `Sendable`을 준수해야 합니다.  

이러한 변경 사항으로 인해 새로운 제한 사항은 Swift 6 모드에서만 적용되며, Swift 5 및 이전 버전에서는 경고로 표시됩니다.  
## Effect on API resilience
이 제안은 API 복원력에 영향을 미치지 않습니다!
## Alternatives Considered
이 제안과 관련하여 논의할 만한 몇 가지 대안이 있습니다. 여기서는 주요한 몇 가지를 소개합니다.  
### Exotic Type System Features
[Swift concurrency road map](https://forums.swift.org/t/swift-concurrency-roadmap/41611)에서는 향후 기능 세트의 반복에 "`mutableIfUnique`" class와 같은 새로운 타입 시스템 기능이 도입될 수 있다고 언급하고 있으며, 이동 의미론과 고유 소유권이 언젠가 Swift에 도입될 수 있다고 쉽게 상상할 수 있습니다.  

향후 제안의 전체 사양을 알지 못하면 세부적인 상호 작용을 이해하기 어렵지만, `Sendable` 검사를 시행하는 검사 기계는 간단하고 구성이 가능하다고 생각합니다. 동시성 경계를 넘어 안전하게 전달할 수 있는 모든 유형에서 작동해야 합니다.   
### Support an explicit copy hook
[이 제안의 첫 번째 개정안](https://docs.google.com/document/d/1OMHZKWq2dego5mXQtWt1fm-yMca2qeOdCl8YlBG1uwg/edit#)에서는 `unsafeSend` protocol 요건을 구현하여 동시성 도메인을 가로질러 전송될 때 타입이 사용자 정의 동작을 정의할 수 있도록 했습니다. 이로 인해 제안의 복잡성이 증가하고, 원치 않는 기능(명시적으로 구현된 복사 동작)이 인정되었으며, 재귀 집계 케이스가 더 비싸고 코드 크기가 더 커졌습니다.    
## Conclusion
이 제안은 동시성 도메인 간에 안전하게 전송할 수 있는 유형을 정의하는 매우 간단한 접근 방식을 정의합니다. 기존 Swift 기능과 일관되고, 사용자가 확장할 수 있으며, 레거시 코드 베이스에서 작동하고, 20년 후에도 만족할 수 있는 간단한 모델을 제공하는 최소한의 컴파일러/언어 지원만 필요합니다.  
  
이 기능은 대부분 기존 언어 지원을 기반으로 하는 라이브러리 기능이기 때문에 도메인별 문제에 맞게 확장하는 래퍼 유형을 쉽게 정의할 수 있으며(위의 `NSCopied` 예시와 같이), 소급 준수를 통해 사용자가 아직 업데이트되지 않은 이전 라이브러리로도 Swift Concurrency 모델에 대해 쉽게 작업할 수 있습니다.  
## Revision history
* 두 번째 리뷰에서 변경된 사항
  * 리뷰 피드백 및 코어 팀 결정에 따라 `@sendable`을 `@Sendable`로 이름 변경.  
  * Marker protocol에 대한 향후 방향을 사용자 정의 속성으로 추가.  
  * 고려되는 대안에서 "Swift 동시성 1.0" 및 "2.0" 논의 삭제.  
* 첫 번째 검토에서 변경된 사항  
  * `ConcurrentValue`를 `Sendable`로, `@concurrent`를 `@sendable`로 이름 변경.  
  * `UnsafeConcurrentValue`를 `@unchecked Sendable` 적합성으로 대체.  
  * non-public, non-frozen `struct` 및 `enum` 유형에 대해 `Sendable`에 암시적 적합성을 추가했습니다.

참고: [Concurrent-value-and-concurrent-closures](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
