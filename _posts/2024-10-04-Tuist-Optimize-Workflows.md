---
layout: post
title: [Tuist] Optimize workflows
date: "2024-10-04 00:00:00 +0900"
categories: ["Tuist"]
tags:
- tuist
- iOS
- guide
type: post
published: true
meta: {}
---
Tuist는 설명과 수집한 인사이트를 통해 프로젝트를 파악하기 때문에 워크플로우를 최적화하여 더 효율적으로 만들 수 있습니다. 몇 가지 예를 살펴봅시다.
## Smart test runs
`tuist test`를 다시 실행해 보겠습니다. 다음과 같은 메시지가 표시됩니다:
```bash
There are no tests to run, finishing early
```
마지막으로 테스트를 실행한 이후 프로젝트에서 변경한 사항이 없으므로 테스트를 다시 실행할 필요가 없습니다. 무엇보다도 이 기능은 다양한 머신과 CI 환경에서 작동한다는 점이 가장 좋습니다.
## Cache
일반적으로 CI에서 또는 암호화 컴파일 문제를 해결하기 위해 전역 캐시를 정리한 후 프로젝트를 클린 빌드하는 경우 전체 프로젝트를 처음부터 컴파일해야 합니다. 프로젝트의 규모가 커지면 시간이 오래 걸릴 수 있습니다.

Tuist는 이전 빌드의 바이너리를 재사용하여 이 문제를 해결합니다. 다음 명령을 실행합니다:
```bash
tuist cache  
```
이 명령은 프로젝트에서 캐시 가능한 모든 대상을 로컬 및 원격 캐시에 빌드하고 공유합니다. 완료되면 프로젝트를 생성해 보세요:
```bash
tuist generate
```
프로젝트 그룹에 캐시의 바이너리가 포함된 새 그룹 `Cache`가 포함된 것을 확인할 수 있습니다.

변경 사항을 원격 리포지토리에 업스트림으로 푸시하면 다른 개발자가 프로젝트를 복제하고 다음 명령을 실행할 수 있습니다:
```bash
tuist install  
tuist auth  
tuist generate
```
그러면 갑자기 종속성이 바이너리로 포함된 프로젝트를 얻을 수 있습니다.
## Optimizations on CI
CI에서 이러한 최적화에 액세스하려면 CI 환경에서 요청을 인증하기 위해 프로젝트 범위 토큰을 생성해야 합니다.
```bash
tuist project tokens create my-handle/MyApp
```
그런 다음 토큰을 CI 환경에서 환경 변수 `TUIST_CONFIG_TOKEN`으로 노출합니다. 토큰이 있으면 자동으로 최적화와 인사이트가 활성화됩니다.

>CI 환경 감지  
Tuist는 CI 환경에서 실행 중임을 감지한 경우에만 토큰을 사용합니다. CI 환경이 감지되지 않는 경우 환경 변수 `CI`를 `1`로 설정하여 토큰 사용을 강제할 수 있습니다.
