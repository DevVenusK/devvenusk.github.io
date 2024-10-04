---
layout: post
title: Tuist - Create Project
date: "2024-10-01 00:00:00 +0900"
categories: ["iOS/Tuist"]
tags:
- tuist
- iOS
- guide
type: post
published: true
meta: {}
---
Tuist를 설치한 후에는 다음 명령을 실행하여 새 프로젝트를 만들 수 있습니다:
```bash
mkdir MyApp
cd MyApp
tuist init --name MyApp
```
기본적으로 **iOS 애플리케이션**을 나타내는 프로젝트가 생성됩니다. 프로젝트 디렉터리에는 프로젝트를 설명하는 `Project.swift`, 프로젝트 범위의 Tuist 구성이 포함된 `Tuist/Config.swift`, 애플리케이션의 소스 코드가 포함된 `MyApp/` 디렉터리가 포함됩니다.

Xcode에서 작업하려면 Xcode 프로젝트를 실행하여 생성할 수 있습니다:
```bash
tuist generate
```
직접 열고 편집할 수 있는 Xcode 프로젝트와 달리 Tuist 프로젝트는 매니페스트 파일에서 생성됩니다. 즉, 생성된 Xcode 프로젝트를 직접 편집해서는 안 됩니다.
>충돌이 없고 사용자 친화적인 환경  
Xcode 프로젝트는 충돌이 발생하기 쉽고 사용자에게 많은 복잡성을 노출합니다. Tuist는 특히 프로젝트의 종속성 그래프 관리 영역에서 이를 추상화합니다.
## Build the app
Tuist는 프로젝트에서 수행해야 하는 가장 일반적인 작업에 대한 명령을 제공합니다. 앱을 빌드하려면
```bash
tuist build
```
이 명령은 내부적으로 플랫폼의 빌드 시스템(예: `xcodebuild`)을 사용하여 Tuist의 기능을 강화합니다.
## Test the app
마찬가지로 다음을 사용하여 테스트를 실행할 수 있습니다:
```bash
tuist test
```
`build` 명령과 마찬가지로 `test`는 플랫폼의 테스트 러너(예: `xcodebuild test`)를 사용하지만 Tuist의 테스트 기능 및 최적화의 이점이 추가됩니다.

>기본 빌드 시스템에 인수 전달하기  
`build`와 `test` 모두 `--` 뒤에 추가 인수를 받아 기본 빌드 시스템으로 전달할 수 있습니다.
