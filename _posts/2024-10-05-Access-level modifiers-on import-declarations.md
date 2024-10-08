---
layout: post
title: Swift Evolution - Access-level modifiers on import declarations
date: "2024-10-05 00:00:00 +0900"
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
- **Proposal**: SE-0409
- **Author**: Alexis Laferrière
- **Review Manager**: Frederick Kellison-Linn
- **Status**: Implemented (Swift 6.0)
- **Implementation**: On main and release/5.9 gated behind the frontend flag `-enable-experimental-feature AccessLevelOnImport`
- **Upcoming Feature**: InternalImportsByDefault

## Introduction
import 선언에 액세스 수준 수정자를 사용하여 종속성의 가시성을 선언하면 가져온 모듈을 참조할 수 있는 선언을 강제할 수 있습니다. 종속성은 소스 파일, 모듈, 패키지 또는 모든 클라이언트에게만 보이도록 표시할 수 있습니다. 이렇게 하면 종속성 및 가져온 선언에 선언의 액세스 수준이라는 익숙한 동작이 적용됩니다. 이 기능은 클라이언트에서 구현 세부 정보를 숨길 수 있으며 종속성 크리프를 관리하는 데 도움이 됩니다.
## Motivation
모범 사례에서는 외부 클라이언트가 내부 세부 정보에 의존하지 않도록 공용 서비스와 내부 서비스를 분리하도록 권장합니다. Swift는 이미 타입 검사 중에 선언과 시행에 대한 각각의 수정자를 가진 접근 수준을 제공합니다, 하지만 현재 종속성에 대한 동등한 공식 기능은 없습니다.

라이브러리 작성자는 각 라이브러리 종속성에 대해 다른 의도를 가지고 있을 수 있습니다. 일부는 라이브러리 클라이언트에게 알려야 하는 반면, 일부는 패키지, 모듈 또는 소스 파일 내부의 구현 세부 사항을 위한 것입니다. 종속성의 의도된 액세스 수준을 강제할 방법이 없으면 구현 세부 사항으로 유지하려는 경우에도 공개 선언에서 참조하여 라이브러리의 종속성을 라이브러리 클라이언트에 노출하는 실수를 저지르기 쉽습니다.

또한 모든 라이브러리 종속성이 라이브러리 클라이언트에 표시되면 컴파일러가 필요 이상으로 많은 작업을 수행해야 합니다. 컴파일러는 라이브러리의 클라이언트를 빌드할 때 모든 라이브러리 종속성을 로드해야 합니다,  
실제로 클라이언트를 빌드하는 데 필요하지 않은 종속성까지 모두 로드해야 합니다.
## Proposed solution
이 제안의 핵심은 현재 접근 수준 로직을 확장하여 import 선언에서 기존 수정자(`open` 제외)를 선언하고  
가져온 선언에 접근 수준을 적용하는 것입니다.

다음은 로컬 모듈의 구현 세부 사항인 모듈 `DatabaseAdapter`의 예시입니다. 이 모듈을 클라이언트에 노출하고 싶지 않으므로 `import internal`로 표시합니다. 그러면 컴파일러는 내부 함수에서 참조를 허용하지만 공용 함수의 시그니처에서 참조를 진단합니다.
```swift  
internal import DatabaseAdapter

internal func internalFunc() -> DatabaseAdapter.Entry {...} // Ok
public func publicFunc() -> DatabaseAdapter.Entry {...} // error: function cannot be declared public because its result uses an internal type
```  
또한 이 제안은 모듈을 구성하는 모든 소스 파일의 각 import 선언에 선언된 접근 수준을 사용하여 라이브러리의 클라이언트가 라이브러리의 종속성을 로드해야 하는 시기 또는 건너뛸 수 있는 시기를 결정합니다. 소스 호환성과 모범 사례의 균형을 맞추기 위해 명시적 액세스 수준이 없는 import는 Swift 5 및 Swift 6에서 암시적 액세스 수준인 `public`을 갖습니다. 향후 언어 모드에서는 `internal`가 될 것입니다. import의 `@usableFromInline` 속성은 인라인 가능한 코드에서 참조를 허용합니다.
## Detailed design
이 섹션에서는 이 제안의 세 가지 주요 언어 변경 사항에 대해 설명합니다. import 선언에 접근 수준 수정자를 허용하여 import된 모듈의 가시성을 선언합니다. 소스 파일을 타입 검사할 때 해당 정보를 적용하고, 그리고 간접 클라이언트가 전이 종속성 로드를 건너뛸 수 있는 시기를 결정합니다. 그런 다음 이 제안에서 다루는 다른 문제를 다룹니다. 언어 모드에 따라 import의 기본 액세스 수준이 달라지는 문제, 그리고 import의 다른 어트리뷰트와의 관계에 대해 설명합니다.
### import된 모듈의 접근 수준 선언하기
접근 수준은 import 선언 앞에 선언에 사용되는 몇 가지 수정자 중 일부를 사용하여 선언합니다: `public`, `package`, `internal`, `fileprivate`, `private`.

공용 종속성은 모든 선언에서 참조할 수 있으며 모든 클라이언트에서 볼 수 있습니다. 공용 종속성은 `public` 수정자와 함께 선언됩니다.
```swift
public import PublicDependency  
```  
같은 패키지의 모듈에만 표시되는 종속성은 `package` 수정자와 함께 선언됩니다. `package`, `internal`, `fileprivate` 및 `private` 선언의 서명만 가져온 모듈을 참조할 수 있습니다.
```swift  
public import PublicDependency
```  
모듈 내부의 종속성은 `internal` 수정자를 사용하여 선언합니다. `internal`, `fileprivate` 및 `private` 선언의 서명만 가져온 모듈을 참조할 수 있습니다.
```swift  
internal import InternalDependency
```  
이 소스 파일에 대한 비공개 종속성은 `fileprivate` 또는 `private` 수정자를 사용하여 선언됩니다. 두 경우 모두 가져오기를 선언하는 소스 파일로 액세스 범위가 지정됩니다. `fileprivate` 및 `private` 선언의 서명만 가져온 모듈을 참조할 수 있습니다.
```swift  
fileprivate import DependencyPrivateToThisFile
private import OtherDependencyPrivateToThisFile
```  
`open` 액세스 수준 수정자는 가져오기 선언에서 거부됩니다.

인라인에서 종속성을 참조할 수 있는 선언 서명을 제한하면서 인라인 가능한 코드에서 종속성을 참조할 수 있도록 하기 위해 `@usableFromInline` 속성을 가져오기 선언에 적용할 수 있습니다. 속성은 `package` 및 `internal` import에만 `@usableFromInline`을 사용할 수 있습니다. 이 속성은 종속성을 클라이언트가 볼 수 있도록 표시합니다.
```swift
@usableFromInline package import UsableFromInlinePackageDependency
@usableFromInline internal import UsableFromInlineInternalDependency
```  
*참고: import에서 `@usableFromInline`에 대한 지원은 아직 구현되지 않았습니다.*
### import된 모듈에 대한 타입 검사 참조
현재 타입 검사는 해당 선언이 각각의 액세스 수준을 존중하도록 강제합니다. 더 잘 보이는 선언이 덜 보이는 선언을 참조하는 경우 오류로 보고합니다. 예를 들어, `public` 함수 서명이 `internal` 유형을 사용하는 경우 오류를 발생시킵니다.

이 제안은 import 선언의 접근 수준을 가져오기와 함께 소스 파일 내에서 imported 선언의 가시성에 대한 상한으로 사용하여 기존 논리를 확장합니다. 예를 들어, `internal import SomeModule`이 있는 소스 파일을 타입 검사할 때, `SomeModule`에서 가져온 모든 선언은 파일 컨텍스트에서 접근 수준이 `internal`인 것으로 간주합니다. 이 경우 유형 검사에서는 `internal`로 imported 선언이 `internal` 이하의 선언 서명과 일반 함수 본문에서만 참조되도록 강제합니다. 공개 선언 서명, `@usableFromInline` 선언 서명 또는 인라인 가능한 코드에는 표시될 수 없습니다. 이는 현재 선언의 액세스 수준 수정자와 인라인 가능한 코드에 적용되는 익숙한 진단에 의해 보고됩니다.

`package`, `fileprivate` 및 `private` import 선언에도 동일한 논리를 적용합니다. `public` import의 경우, imported 선언을 참조할 수 있는 방법에 대한 제한이 없습니다. 공개 선언 서명에서 참조할 수 없는 가져온 `package` 선언에 대한 기존 제한을 넘어서는 것입니다.

인라인 코드에 대한 import의 `@usableFromInline` 속성은 인라인 가능한 코드에 적용됩니다. 인라인 가능한 코드: `@inlinable` 및 `@backDeployed` 함수 본문, 인수의 기본 `initalizer`, `@frozen` 구조체의 프로퍼티에 적용됩니다. 인라인으로 가져온 `@usableFromInline` 종속성은 인라인 가능한 코드에서 참조할 수 있지만 액세스 수준만 고려되는 선언 서명의 유형 검사에는 영향을 미치지 않습니다.

다음은 `fileprivate` import를 사용하는 일반적인 경우의 유형 검사에서 생성된 대략적인 진단의 예입니다.
```swift  
fileprivate import DatabaseAdapter  
  
fileprivate func fileprivateFunc() -> DatabaseAdapter.Entry { ... } // Ok  
  
internal func internalFunc() -> DatabaseAdapter.Entry { ... } // 오류: 반환값이 fileprivate 타입을 사용하므로 함수를 내부로 선언할 수 없습니다.  
  
public func publicFunc(entry: DatabaseAdapter.Entry) { ... } // 오류: 매개변수가 파일 비공개 형식을 사용하기 때문에 함수를 공개로 선언할 수 없습니다.  
  
public func useInBody() {  
	DatabaseAdapter.create() // Ok  
}  
  
@inlinable  
public func useInInlinableBody() {  
	DatabaseAdapter.create() // 오류: 전역 함수 'create()'는 파일 비공개이며 '@inlinable' 함수에서 참조할 수 없습니다.  
}  
```  
### 전이 종속성 로딩
모듈 수준에서 이 액세스 수준 정보를 사용할 때 종속성을 공개적으로 가져온 적이 없고 다른 요구 사항이 충족되면 클라이언트에서 종속성을 숨길 수 있습니다. 그러면 전이 종속성을 로드하지 않고도 클라이언트를 빌드할 수 있습니다. 이렇게 하면 빌드 시간이 단축되고 구현 세부 사항인 모듈을 배포할 필요가 없습니다.

동일한 종속성을 동일한 모듈의 다른 파일에서 다른 액세스 수준으로 가져올 수 있습니다. 모듈 수준에서는 가장 허용되는 액세스 수준만 고려합니다. 예를 들어 종속성을 두 개의 서로 다른 파일에서 `package` 및 `internal`로 가져온 경우 모듈 수준에서 `package` 가시성 종속성으로 간주합니다.

모듈 수준 정보는 전이적 클라이언트에 대해 서로 다른 동작을 의미합니다. 전이적 클라이언트는 모듈에 간접 종속성이 있는 모듈입니다. 예를 들어, 다음 시나리오에서 `TransitiveClient`는 `MiddleModule`을 가져와서 `IndirectDependency`의 전이적 클라이언트입니다.

```  
module IndirectDependency
         ↑
module MiddleModule
         ↑
module TransitiveClient
```  

중간 모듈에서 간접 종속성을 가져오는 방식에 따라 전이적 클라이언트는 컴파일 시 이를 로드할 수도 있고 로드하지 않을 수도 있습니다. 전이 종속성을 로드하는 데 필요한 네 가지 요소가 있으며, 이 중 하나라도 해당되지 않으면 종속성을 숨길 수 있습니다.
1. `public` 또는 `@usableFromInline` 종속성은 항상 전이적 클라이언트에서 로드해야 합니다.
2. 탄력적이지 않은 모듈의 모든 종속성은 전이적 클라이언트에서 로드해야 합니다. 이는 모듈의 타입이 저장소에서 해당 종속성의 타입을 사용할 수 있고 컴파일러가 코드를 올바르게 방출하기 위해 비복원성 타입의 저장소에 대한 완전한 정보가 필요하기 때문입니다. 이 제한에 대해서는 향후 방향 섹션에서 자세히 설명합니다.
3. 모듈과 전이 클라이언트가 동일한 패키지의 일부인 경우 모듈의 `package` 종속성은 해당 전이 클라이언트에서 로드해야 합니다. 이는 모듈의 `package` 선언이 해당 종속성의 유형을 서명에 사용할 수 있기 때문입니다. 패키지 이름이 일치하면 두 모듈이 같은 패키지에 있는 것으로 간주하여 패키지 선언에 사용된 것과 동일한 논리를 적용합니다.
4. 전이적 클라이언트에 `@testable` import가 있는 경우 모듈의 모든 종속성을 로드해야 합니다. 이는 테스트 가능한 클라이언트가 모든 수준의 import 가시성을 가진 종속성에 의존할 수 있는 `internal` 선언을 사용할 수 있기 때문입니다. `private` 및 `fileprivate` 종속성도 로드해야 합니다.

다른 모든 경우에는 종속성이 숨겨지며 전이성 클라이언트에서 로드할 필요가 없습니다. 한 가져오기 경로에 숨겨진 종속성은 다른 가져오기 경로로 인해 로드해야 할 수도 있습니다.

숨겨진 종속성과 관련된 모듈 인터페이스는 클라이언트에 배포할 필요가 없습니다. 그러나 모듈에 연결된 바이너리는 결과 프로그램을 실행하기 위해 여전히 배포해야 합니다.
### 기본 가져오기 액세스 수준
명시적 액세스 수준 수정자가 없는 기본 import 선언의 액세스 수준은 언어 버전에 따라 다릅니다. 여기에서는 암시적 액세스 수준과 이 선택에 대한 이유를 나열합니다.

Swift 6까지의 언어 모드에서는 import가 기본적으로 `public`로 설정됩니다. 이 선택은 소스 호환성을 유지합니다. 이전에 Swift 5에서 사용할 수 있었던 유일한 공식 import는 이 문서에서 제안한 public import와 같이 작동합니다.

향후 언어 모드에서는 import가 기본적으로 `internal` import가 됩니다. 이렇게 하면 import의 동작이 암시적 액세스 수준이 internal인 선언과 일치하게 됩니다. 종속성을 public로 표시하려면 명시적인 수정자가 필요하므로 의도하지 않은 종속성 크리프를 제한하는 데 도움이 될 것입니다.

결과적으로 다음 import는 Swift 6까지의 언어 모드에서는 `pubilc`이지만 향후 언어 모드에서는 `internal`가 됩니다.
```swift  
import ADependency  
```  
향후 언어가 변경되면 새 언어 모드를 채택하는 코드의 소스 변경이 필요할 수 있습니다. 현재 언어 모드에 남아 있는 코드에 대한 소스 호환성은 깨지지 않습니다. 마이그레이션 도구는 필요한 경우 `public` 수정자를 자동으로 삽입할 수 있습니다. 도구를 사용할 수 없는 경우, 간단한 스크립트를 통해 모든 import 앞에 `public` 수정자를 삽입하여 Swift 5 동작을 유지할 수 있습니다.

곧 출시될 기능 플래그 `InternalImportsByDefault`를 사용하면 Swift 5 또는 6을 사용하는 경우에도 향후 언어 동작을 사용할 수 있습니다.
### import 시 다른 수정자와의 상호작용
클라이언트는 가져온 모듈 선언이 로컬 모듈의 일부인 경우에만 볼 수 있으므로 `@_exported` 속성은 `public` import보다 한 단계 위입니다. 이 제안에 따르면 `@_exported`는 현재 언어 모드에서 수정자 또는 기본 public 가시성 모두와 함께 `public` import 선언에만 허용됩니다.

`@testable` 속성을 사용하면 로컬 모듈이 가져온 모듈의 내부 선언을 참조할 수 있습니다. 현재 디자인은 공개 선언에서 가져온 내부 또는 패키지 유형을 사용할 수도 있습니다. 액세스 수준 동작은 일반 import와 동일한 방식으로 적용되며, 모든 import 선언은 import 선언의 액세스 수준을 상한으로 합니다. 테스트 가능한 import의 경우, imported 내부 선언도 바운드의 영향을 받습니다.

현재 `@_implementationOnly` import를 사용하는 경우 internal import 또는 그 이하로 대체해야 합니다. 이에 비해 이 새로운 기능은 더 엄격한 유형 검사를 가능하게 하고 불필요한 경고를 더 적게 표시합니다. internal import로 대체한 후에도 전이 종속성 로드 요구 사항은 복원력 있는 모듈의 경우 동일하게 유지되지만 전이 종속성을 항상 로드해야 하는 비복원력 모듈의 경우 변경됩니다. 모든 경우에 `@_implementationOnly에` 의존하는 모듈을 업데이트하는 대신 internal import를 사용할 것을 강력히 권장합니다.

scoped import 기능은 동일한 import에 선언된 액세스 수준과는 독립적입니다. 아래 예에서 모듈 `Foo`는 모듈 수준에서 public 종속성이며 로컬 소스 파일의 공개 선언 서명에서 참조할 수 있습니다. 범위가 지정된 부분인 구조체 Foo.Bar는 이 파일에서 Bar만 참조할 수 있도록 조회를 제한하고 다른 import에 다른 Bar 선언이 있는 경우 이 Bar에 대한 참조를 우선적으로 해결합니다. scoped import 단일 선언의 액세스 수준을 제한하는 데 사용할 수 없습니다.
```swift
public import 구조체 Foo.Bar  
```  
## 소스 호환성
소스 호환성을 유지하기 위해 Swift 6을 포함한 현재 언어 모드에서는 import가 기본적으로 공개됩니다. 이렇게 하면 Swift 5에서 import의 현재 동작이 유지됩니다. 앞서 설명한 대로 향후 언어 모드 동작은 기본값을 변경하므로 코드를 변경해야 합니다.
## ABI 호환성
이 제안은 ABI 호환성에는 영향을 미치지 않습니다. 타입 검사에 의해 컴파일 시간 변경이 적용됩니다.
## 채택에 따른 영향
이 기능을 채택하거나 되돌리더라도 주의해서 사용한다면 클라이언트에는 영향을 미치지 않습니다. 복원력이 없는 모듈에 적용하는 경우 모듈 소스 파일의 유형 검사에만 변경이 적용됩니다. 이 경우 다른 종속성의 액세스 수준을 변경해도 클라이언트에는 영향을 미치지 않습니다.

복원력이 있는 모듈에서 채택하는 경우, 기존 import를 public 미만으로 표시하면 클라이언트 빌드 방식에 영향을 미칩니다. 컴파일러는 더 적은 수의 전이 종속성을 로드하여 클라이언트를 빌드할 수 있습니다. 이론적으로는 클라이언트에 영향을 미치지 않아야 하지만 여전히 다른 컴파일 동작으로 이어질 수 있습니다.

이론적으로 이러한 전이 종속성은 클라이언트에서 사용할 수 없습니다. 따라서 숨겨도 클라이언트에는 영향을 미치지 않습니다. 실제로는 전이 종속성에서 확장 멤버를 사용할 수 있는 누수가 있습니다. 이 기능을 채택하면 전이 종속성 로드를 건너뛰고 이러한 누수를 방지할 수 있습니다. 이러한 동작에 의존하는 코드의 소스 호환성을 깨뜨릴 수 있습니다.
## 향후 방향
### 복원력이 없는 모듈에 대한 종속성 숨기기
복원력이 없는 모듈에 대한 종속성을 숨기는 것은 이론적으로는 가능하지만 컴파일 프로세스에서 몇 가지 제한 사항을 다시 생각해야 합니다. 가장 큰 제약은 컴파일러가 전이 종속성에 따라 달라질 수 있는 import된 타입의 메모리 레이아웃을 알아야 한다는 점입니다. 탄력적인 모듈은 런타임에 이 정보를 제공할 수 있으므로 빌드 시점에 전이적 모듈이 필요하지 않습니다. 탄력적이지 않은 모듈은 런타임에 이 정보를 제공하지 않으므로 컴파일러가 빌드 시점에 전이 종속성을 로드하여 액세스해야 합니다. 해결책으로는 각 모듈에 필요한 정보를 복사하거나, 또는 종속성을 참조할 수 있는 방법을 더 제한하는 방법이 있습니다. 모든 경우에서 이 제안과는 별개의 기능입니다.
## 고려되는 대안
### `@_implementationOnly import`
비공식 `@_implementationOnly` 속성은 유형 검사 및 전이 종속성 숨기기와 유사한 기능을 제공합니다.  
이 속성은 복원력이 없는 모듈에서 사용하거나 `@testable` import와 함께 사용하면 불안정성과 런타임 충돌을 일으킵니다.이 제안과는 약간 다른 의미를 적용하며 유형 검사가 엄격하지 않습니다. 이 제안은 자체 타입 검사 로직에 의존하여 public 선언에서 구현 전용으로 가져온 모듈에 대한 참조를 보고합니다. 이와 대조적으로 이 제안은 기존의 접근 수준 검사 로직과 시맨틱을 사용합니다. 를 사용하여 더 쉽게 배울 수 있습니다. 또한 이 제안은 `package` import와 `private` 및 `fileprivate`을 사용한 file-scoped import  통해 완전히 새로운 기능을 도입합니다.
### 공식 `@_exported 가져오기`로 `open import` 사용
이 제안은 특정한 의미를 부여하지 않기 때문에 `open`이라는 액세스 수준 수정자를 import에 계속 사용할 수 있습니다. 이를 공식 `@_exported`로 사용하는 것이 제안되었습니다. 즉, 모듈의 모든 소스 파일에서 볼 수 있고 클라이언트에게 동일한 모듈의 일부인 것처럼 표시되는 import를 표시하는 것입니다. 일반적으로 `@_exported`를 사용하여 두 모듈이 같은 이름을 공유하는 모듈을 클랭할 때 을 사용하는데, 두 모듈이 같은 이름을 공유하며 클라이언트에게 통합된 것으로 표시하려는 의도가 있습니다.

이 제안에 이 변경 사항을 포함시키지 않은 이유는 크게 두 가지입니다:
1. `open`으로 표시된 선언은 모듈 외부에서 재정의할 수 있습니다. 이 의미는 `@_exported`의 동작과 관련이 없습니다. 다른 액세스 수준은 선언과 가져오기 선언에서 사용할 때 상응하는 의미를 갖습니다.
2. 이 제안의 동기는 구현 세부 사항을 숨기고 의존성 크리프를 제한하기 위해서입니다. `open import` 또는 `@_exported`의 사용을 장려하는 것은 이러한 동기에 반하며 다른 문제를 해결합니다. 관련 동기를 가진 별도의 제안서에서 논의되어야 합니다.

### API에서의 사용으로부터 의존성의 가시성 추론하기
컴파일러는 모듈을 분석하여 public 선언에서 사용되며 클라이언트에게 표시되어야 하는 종속성을 결정할 수 있습니다. 그런 다음 다른 모든 종속성을 자동으로 내부 종속성으로 간주하고 다른 기준을 충족하면 간접 클라이언트에서 숨길 수 있습니다.

이 접근 방식은 import 선언의 액세스 수준 수정자가 제공하는 정보와 선언 서명의 참조가 중복되지 않습니다.  
이러한 중복을 통해 이 제안에 설명된 유형 검사 동작이 가능합니다. 컴파일러가 import에 표시된 의도와 선언 서명에 사용된 의도를 비교할 수 있게 해줍니다. 이 검사는 종속성이 분산되지 않은 경우에 중요합니다. 숨겨진 종속성에서 공개 종속성으로 변경하면 타사에서 사용할 수 없는 종속성에서 배포된 모듈이 손상될 수 있습니다.
## 감사의 말
Becca Royal-Gordon은 이 제안의 설계에 기여하고 프리피치를 작성했습니다.

원본: [Access-level modifiers on import declarations](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0418-inferring-sendable-for-methods.md)
