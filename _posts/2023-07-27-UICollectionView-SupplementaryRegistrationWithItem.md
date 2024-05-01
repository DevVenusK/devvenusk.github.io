---
layout: post
title: Item 을 가진 UICollectionView.SupplementaryRegistration
date: "2024-07-27 00:00:00 +0900"
categories: ["iOS/tech"]
tags:
- iOS
- tech
- swift
type: post
published: true
meta: {}
---
`iOS14` 부터 지원하는 `UICollectionView` API 중 `struct CellRegistration<Cell, Item> where Cell : UICollectionViewCell`의 경우 `Cell`과 `Item`을 전달하게 되는데 `struct SupplementaryRegistration<Supplementary> where Supplementary : UICollectionReusableView`의 경우 `Supplementary`만 전달하게 된다. 아래와 같이 확장하면 `Supplementary`와 `Item`을 전달 할 수 있다.    
```swift
extension UICollectionView { 
  public struct SupplementaryRegistrationWithItem<Supplementary, Item> where Supplementary: UICollectionReusableView {

      public typealias Handler = (
        Supplementary,
        String,
        IndexPath,
        Item
      ) -> Void

      public let elementKind: String
      public let handler: Handler
      public let supplementaryRegistration: SupplementaryRegistration<Supplementary>
  
      public init(
        elementKind: String,
        handler: @escaping Handler
      ) {
        self.elementKind = elementKind
        self.handler = handler
        self.supplementaryRegistration = .init(elementKind: elementKind, handler: { _, _, _   in })
      }
    }

    /// Dequeues a configured reusable supplementary view object.
    ///  - Parameters:
    ///     - registration: The supplementary registration for configuring the supplementary view object. See UICollectionView.SupplementaryRegistration.
    ///     - indexPath: The index path that specifies the location of the supplementary view in the collection view.
    ///     - item: The item that provides data for the supplementary.
    @MainActor
    func dequeueConfiguredReusableSupplementary<Supplementary, Item>(
      using registration: SupplementaryRegistrationWithItem<Supplementary, Item>,
      for indexPath: IndexPath,
      item: Item
    ) -> Supplementary where Supplementary: UICollectionReusableView {
      let supplementaryView = self.dequeueConfiguredReusableSupplementary(
        using: registration.supplementaryRegistration,
        for: indexPath
      )
  
      registration.handler(
        supplementaryView,
        registration.elementKind,
        indexPath,
        item
      )
      return supplementaryView
    }
}
```
### 참고
[UICollectionView.CellRegistration](https://developer.apple.com/documentation/uikit/uicollectionview/cellregistration)    
[UICollectionView.SupplementaryRegistration](https://developer.apple.com/documentation/uikit/uicollectionview/supplementaryregistration)     
