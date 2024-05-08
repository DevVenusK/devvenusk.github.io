---
layout: post
title: NSCache의 생명주기를 관리하기
date: "2024-02-12 00:00:00 +0900"
categories: ["iOS/tech"]
tags:
- iOS
- tech
- swift
type: post
published: true
meta: {}
---
`NSCache`를 사용하다보면 시스템에 의해 의도치 않게 `cache`가 삭제되는 경우가 있다. 이럴경우 `protocol NSDiscardableContent`를 이용해 `cache`삭제에 대한 정책을 정의할 수 있다.   
```swift
import Foundation
final class LottieAnimationCacheObject: NSObject,
NSDiscardableContent {
    internal var animation: LottieAnimation?
    internal func beginContentAccess() -> Bool {
        return true
    ｝

    internal func endContentAccess() {}

    internal func discardContentIfPossible() {
        animation = nil
    ｝

    internal func isContentDiscarded() -> Bool {
        return animation == nil
    ｝
}
```   
### 참고
[protocol NSDiscardableContent](https://developer.apple.com/documentation/foundation/nsdiscardablecontent)   
[NSCache instead of LRUCache for LottieView animation would restart from beginning after backgrounding app](https://github.com/airbnb/lottie-ios/pull/2290)
