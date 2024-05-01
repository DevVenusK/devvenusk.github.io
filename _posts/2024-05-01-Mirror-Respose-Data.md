---
layout: post
title: Mirror를 이용해 API Response를 처리해보자
date: "2024-05-01 14:57:00 +0900"
categories: ["iOS/tech"]
tags:
- iOS
- tech
- swift
type: post
published: true
meta: {}
---
`API Response`에서 `action`값의 유무에 따라 다른 행동을 해야한다면 ‘init(from decoder: any Decoder) throws`가 아닌 `Mirror`를 통해 쉽게 해결할 수 있다.    
```
{
“action1”: “some action1”,
“action2”: “some action2”,
“action3”: “some action3”
}
```    
```swift
struct Response: Decodable {
    let action1: String?
    let action2: String?
    let action3: String?
```    
```swift
let decoded = JSONDecoder().decode(Response.self, from: responseData)
let mirror = Mirror(reflection: decoded)
let doAction = mirror.children.compactMap { $0.value }.first
```    
### 기타
`reflection API`는 `Swift`와 `C++`로 구현되어있다. 
```swift
internal init(internalReflecting subject: Any,
              subjectType: Any.Type? = nil,
              customAncestor: Mirror? = nil)
  {
    let subjectType = subjectType ?? _getNormalizedType(subject, type: type(of: subject))
    
    let childCount = _getChildCount(subject, type: subjectType)
    let children = (0 ..< childCount).lazy.map({
      getChild(of: subject, type: subjectType, index: $0)
    })
    self.children = Children(children)
    
    self._makeSuperclassMirror = {
      guard let subjectClass = subjectType as? AnyClass,
            let superclass = _getSuperclass(subjectClass) else {
        return nil
      }
      
      // Handle custom ancestors. If we've hit the custom ancestor's subject type,
      // or descendants are suppressed, return it. Otherwise continue reflecting.
      if let customAncestor = customAncestor {
        if superclass == customAncestor.subjectType {
          return customAncestor
        }
        if customAncestor._defaultDescendantRepresentation == .suppressed {
          return customAncestor
        }
      }
      return Mirror(internalReflecting: subject,
                    subjectType: superclass,
                    customAncestor: customAncestor)
    }
```   
### 참고
[How Mirror Works](https://www.swift.org/blog/how-mirror-works/)    
[Mirror Source](https://github.com/apple/swift/blob/main/stdlib/public/core/Mirror.swift)     
[Mirror](https://developer.apple.com/documentation/swift/mirror)