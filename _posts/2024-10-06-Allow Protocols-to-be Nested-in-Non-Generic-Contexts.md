---
layout: post
title: Swift Evolution - Access-level modifiers on import declarations
date: "2024-10-06 00:00:00 +0900"
categories: ["swift/evolution/proposals"]
tags:
- swift
- evolution
- proposals
- access-level
type: post
published: true
meta: {}
---
- **Proposal**: SE-0404
- **Author**: Karl Wagner
- **Review Manager**: Holly Borla
- **Status**: Implemented (Swift 5.10)
- **Implementation**: apple/swift#66247 (gated behind flag -enable-experimental-feature NestedProtocols)

## Introduction
프로토콜을 non-generic `struct/class/enum/actor` 및 함수 안에 중첩할 수 있습니다.
## Motivation
normal types을 다른 normal types 안에 중첩하면 개발자가 내부 유형의 자연스러운 범위를 표현할 수 있습니다.(예: `String.UTF8View`는 구조체 `String` 안에 중첩된 구조체 `UTF8View`이며, 그 이름은 문자열 값의 UTF-8 코드 단위 인터페이스라는 목적을 명확하게 전달합니다.

그러나 현재 중첩은 다른 struct/class/enum/actor 내의 struct/class/enum/actor로 제한되며 프로토콜은 전혀 중첩할 수 없으므로 항상 모듈 내에서 최상위 유형이어야 합니다. 이는 안타까운 일이며, 개발자가 자연스럽게 외부 유형으로 범위가 지정된 프로토콜을 표현할 수 있도록 이 제한을 완화해야 합니다.
## Proposed solution
non-generic struct/class/enum/actors 그리고 제네릭 컨텍스트에 속하지 않는 함수 내에서도 프로토콜 중첩을 허용할 것입니다.

예를 들어, `TableView.Delegate`는 당연히 테이블 뷰와 관련된 델리게이트 프로토콜입니다. 개발자는 이를 `TableView` 클래스 내에 중첩하여 선언해야 합니다.
```swift
class TableView {
  protocol Delegate: AnyObject {
    func tableView(_: TableView, didSelectRowAtIndex: Int)
  }
}

class DelegateConformer: TableView.Delegate {
  func tableView(_: TableView, didSelectRowAtIndex: Int) {
    // ...
  }
}
```
현재 개발자들은 중첩을 통해 표현할 수 있는 것과 동일한 자연스러운 범위를 표현하기 위해 `TableViewDelegate와` 같은 복합 이름을 부여하고 있습니다.

추가적인 이점으로, `TableView`의 컨텍스트 내에서 중첩된 프로토콜 `Delegate`는 다른 모든 중첩된 유형과 마찬가지로 더 짧은 이름으로 참조할 수 있습니다.(다른 모든 중첩된 유형의 경우와 마찬가지로)
```swift
class TableView {
  weak var delegate: Delegate?
  
  protocol Delegate { /* ... */ }
}
```
프로토콜은 일반적이지 않은 함수 및 클로저 내에 중첩될 수도 있습니다. 물론 이러한 프로토콜에 대한 모든 적합성 또한 동일한 함수 내에 있어야 하므로 다소 제한적인 유용성이 있습니다. 그러나 개발자가 함수 내에서 생성하는 모델의 복잡성을 인위적으로 제한할 이유도 없습니다. 일부 코드베이스(특히 Swift 컴파일러 자체)는 중첩된 유형이 있는 대형 클로저를 사용하며 프로토콜을 사용한 추상화의 이점을 누리고 있습니다.
```swift
func doSomething() {

   protocol Abstraction {
     associatedtype ResultType
     func requirement() -> ResultType
   }
   struct SomeConformance: Abstraction {
     func requirement() -> Int { ... }
   }
   struct AnotherConformance: Abstraction {
     func requirement() -> String { ... }
   }
   
   func impl<T: Abstraction>(_ input: T) -> T.ResultType {
     // ...
   }
   
   let _: Int = impl(SomeConformance())
   let _: String = impl(AnotherConformance())
}
```
## Detailed design
프로토콜은 일반 컨텍스트를 제외하고 struct/class/enum/actor가 중첩될 수 있는 모든 곳에 중첩될 수 있습니다. 예를 들어 다음은 금지되어 있습니다.
```swift
class TableView<Element> {

  protocol Delegate {  // Error: protocol 'Delegate' cannot be nested within a generic context.
    func didSelect(_: Element)
  }
}
```
제네릭 함수에도 동일하게 적용됩니다.
```swift
func genericFunc<T>(_: T) {
  protocol Abstraction {  // Error: protocol 'Abstraction' cannot be nested within a generic context.
  }
}
```
그리고 제네릭 컨텍스트 내의 다른 함수에도 적용됩니다.
```swift
class TableView<Element> {
  func doSomething() {
    protocol MyProtocol {  // Error: protocol 'Abstraction' cannot be nested within a generic context.
    }
  }
}
```
이를 지원하려면 둘 중 하나가 필요합니다.
- 제네릭 프로토콜을 도입하거나
- 제네릭 매개변수를 연관된 유형에 매핑.
  두 가지 모두 이 제안의 범위에는 포함되지 않지만, 이 작성자는 제네릭 컨텍스트를 지원하지 않아도 충분한 이점이 있다고 생각합니다. 이 중 어느 쪽이든 향후 흥미로운 방향이 될 것입니다.
  
### Associated Type matching
구체적인 유형에 중첩된 경우 프로토콜은 associated type 요구 사항을 확인하지 않습니다.
```swift
protocol Widget {
  associatedtype Delegate
}

struct TableWidget: Widget {
  // Does NOT witness Widget.Delegate
  protocol Delegate { ... }
}
```
Associated type은 하나의 구체적인 타입을 하나의 준수 타입과 연관시킵니다. 프로토콜은 많은 구체적인 유형이 준수할 수 있는 제약 조건 유형이므로 프로토콜이 associated type 요구 사항을 감시하는 데는 분명한 의미가 없습니다.

과거에 프로토콜에 이러한 제약 조건 유형 네트워크를 표현할 수 있는 “associated protocol” 기능을 추가할 수 있는지에 대한 논의가 있었습니다. 만약 그러한 기능이 도입된다면, 오늘날 중첩된 콘크리트 타입이 associated protocol 요구 사항을 감시하는 것과 같은 방식으로 associated protocol 요구 사항이 중첩된 프로토콜에 의해 감시되는 것이 합리적일 수 있습니다.
## Source compatibility
이 기능은 추가 기능입니다.
## ABI compatibility
이 제안은 순전히 언어의 ABI를 확장한 것이며 기존 기능을 변경하지 않습니다.
## Implications on adoption
이 기능은 배포 제약 없이 소스 코드에서 자유롭게 채택하거나 채택하지 않을 수 있습니다.

일반적으로 상위 컨텍스트에서 프로토콜을 안팎으로 이동하는 것은 소스를 깨는 변경입니다. 그러나 새 이름에 `typealias` 별칭을 제공하면 이러한 중단을 완화할 수 있습니다.

다른 중첩된 유형과 마찬가지로, 부모 컨텍스트는 중첩된 프로토콜의 망글링된 이름의 일부를 형성합니다. 따라서 상위 컨텍스트에서 프로토콜을 안팎으로 이동하는 것은 ABI와 호환되지 않는 변경입니다.
## Future directions
- 프로토콜에 다른(프로토콜이 아닌) 타입 중첩 허용
  프로토콜 자체에서 해당 프로토콜로 자연스럽게 범위가 지정된 타입을 정의하고자 하는 경우가 있습니다. 예를 들어, 표준 라이브러리의 [`FloatingPointRoundingRule`](https://developer.apple.com/documentation/swift/FloatingPointRoundingRule) 열거형은 [`FloatingPoint` 프로토콜의 요구 사항에](https://developer.apple.com/documentation/swift/floatingpoint/round(_:)) 의해 사용되며 해당 목적으로 정의됩니다.
- 제네릭 타입에서 프로토콜 중첩 허용
  세부 설계 섹션에서 언급했듯이 제네릭 타입 내에서 프로토콜 중첩을 허용하는 전략이 잠재적으로 있으며, 이러한 표현 기능을 사용하는 방법을 확실히 상상할 수 있습니다. 커뮤니티는 별도의 주제에서 잠재적인 접근 방식에 대해 논의할 수 있습니다.
  
## Alternatives considered
없음. 이는 언어의 기존 중첩 기능의 간단한 확장입니다.
## Acknowledgments
이 사실을 알려주신 [`@jumhyn`](https://forums.swift.org/u/jumhyn/) 님과 [`@suyashsrijan`](https://forums.swift.org/u/suyashsrijan/) 님께 감사드립니다.

참고: [Allow Protocols to be Nested in Non-Generic Contexts](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0404-nested-protocols.md)
