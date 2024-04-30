---
layout: post
title: Swift에서 옵셔널이란 무엇이며, 언제 사용해야 하나요?
date: "2024-04-30 00:00:00 +0900"
categories: ["iOS/interview"]
tags:
- iOS
- interview
type: post
published: true
meta: {}
---
1. 옵셔널 바인딩과 강제 언래핑의 차이점은 무엇인가요?
  - `Optional Binding` (Optional Chaining)
    - 각종 `Binding` 기법을 사용해 값의 유무를 확인한 후 그 값을 할당한다.
  - `Forced Unwrapping` 
    - `!` 함수를 이용해 값의 유무와 상관없이 값을 할당한다.
    - 값이 `nil` 일 경우 runtime crash 가 발생한다.
    - `unsafelyUnwrapped` 와 완벽하게 동일하게 동작한다.
```swift
@inlinable
public var unsafelyUnwrapped: Wrapped {
  @inline(__always)
  get {
    if let x = self {
      return x
    }
    _debugPreconditionFailure("unsafelyUnwrapped of nil optional")
  }
}
```  
2. 옵셔널 체이닝의 동작 원리를 설명해주세요.
  - `?` 메소드를 통해 `Optional Chaining` 을 진행하게 되며 해당 메소드마다 값의 유무를 확인하게 된다. Chaining 과정 중 값이 하나라도 없을 시 `nil` 을 반환한다. 
3. 암시적 언래핑 옵셔널을 사용하는 경우는 언제인가요?
  - Interface Builder 를 통해 생성된 Property 에서 사용한다.
    - 해당 Property 들은 초기화 시점에 할당된다.
  - 객체에서 initialize 시점에 설정되는 Property 에서 사용 가능하다.
4. `nil` 병합 연산자(`??`)의 사용 예시를 들어주세요.
  - 값이 잇다면 `value` 를 return 하고, 없다면 `defaultValue` 를 return 한다.
```swift 
@_transparent
@_alwaysEmitIntoClient
public func ?? <T: ~Copyable>(
  optional: consuming T?,
  defaultValue: @autoclosure () throws -> T // FIXME: typed throw
) rethrows -> T {
  switch consume optional {
  case .some(let value):
    return value
  case .none:
    return try defaultValue()
  }
}
```

### 기타
`optioanl` 의 `map`
```swift
@_alwaysEmitIntoClient
public func map<E: Error, U: ~Copyable>(
  _ transform: (Wrapped) throws(E) -> U
) throws(E) -> U? {
  switch self {
  case .some(let y):
    return .some(try transform(y))
  case .none:
    return .none
  }
}
```
`optioanl` 의 `flatMap`
```swift
@_alwaysEmitIntoClient
public func flatMap<E: Error, U: ~Copyable>(
  _ transform: (Wrapped) throws(E) -> U?
) throws(E) -> U? {
  switch self {
  case .some(let y):
    return try transform(y)
  case .none:
    return .none
  }
}
```
