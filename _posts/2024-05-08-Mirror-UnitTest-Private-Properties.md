---
layout: post
title: Mirror를 이용한 Private Properties를 Test
date: "2024-05-08 23:20:00 +0900"
categories: ["iOS/tech"]
tags:
- iOS
- tech
- swift
- test
- unitTest
type: post
published: true
meta: {}
---
`ReactorKit`과 같은 `Binding`을 이용한 아키텍쳐에서는 `Unit Test`를 작성할 때 `Stream`의 끝은 `UIBinding`일 경우가 많기때문에 이럴경우는 해당 메소드의 값이 `View Property`에 바인딩이 잘 되었는지 테스트를 해야한다고 생각한다. 하지만 대부분의 `UI Property`의 경우 `private`접근자로 선언되어 있어 `XCTest`에서 접근이 불가하다. 이럴 경우 `UI Test`를 사용하지 않아도 `Unit Test`와 `Mirror`를 이용해 간단하게 테스트를 해볼 수 있다.     
```swift
final class ViewController: UIViewController { 
  private let titleLabel: UILabel
  ...
  
  func bind() { 
    viewModel.text
    .bind(to: titleLabel.rx.text)
    .disposed(by: disposeBag)
  }
}
```    
```swift
final class ViewModel { 
  public var textStream: PublisedSubject<String> = .init()
  
  public func chanedText(_ text: String) { 
    textStream.onNext(text)
  }
}
```    
```swift
final class ViewModelTests { 
  func test_ViewModel_TextStream() { 
    viewModel = ViewModel()
    
    let viewController = ViewController(viewModel: viewModel)
    let mirror = Mirror(reflecting: viewController)
    
    // RxText 는 생략
    viewModel.chanedText("chaned Text")
    
    let text = mirror
      .children
      .filter { $0.label == "titleLabel" }
      .map { $0.value }
      .compactMap { $0 as? UILabel }
      .map { $0.text }
      .first!
    
    XCTAssetEqual(text, "changed Text)
  }
}
```
