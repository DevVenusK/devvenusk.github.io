---
layout: post
title: Tuist - Quick Start
date: "2024-09-29 00:00:00 +0900"
categories: ["iOS/Tuist"]
tags:
- tuist
- iOS
- guide
type: post
published: true
meta: {}
---
## Tuist

앱 개발의 세계에서, 특히 Apple과 같은 플랫폼의 경우 조직은 종종 **생산성 장애물**에 직면하게 됩니다. 여기에는 느린 컴파일 시간, 신뢰할 수 없는 테스트, 리소스를 소모하는 복잡한 자동화 워크플로우 등이 포함될 수 있습니다. 일반적으로 기업은 플랫폼 전담 팀을 구성하여 이러한 문제를 해결합니다. 이러한 전문가는 코드베이스의 상태와 무결성을 유지하여 다른 개발자가 기능 개발에 집중할 수 있도록 합니다. 하지만 이러한 접근 방식은 핵심 팀원이 이탈하면 생산성에 심각한 영향을 미칠 수 있기 때문에 비용이 많이 들고 위험할 수 있습니다.

**Tuist는 앱 개발을 가속화하고 향상시키기 위해 설계된 툴체인입니다.** 공식 도구 및 시스템과 원활하게 통합되어 익숙한 영역에서 개발자를 만날 수 있습니다. 도구와 시스템 통합의 부담을 덜어줌으로써 팀이 기능 개발과 전반적인 개발자 경험 개선에 에너지를 쏟을 수 있도록 지원합니다. 본질적으로 Tuist는 가상 플랫폼 팀의 역할을 합니다. 앱 아이디어의 발상부터 사용자 출시까지 모든 단계에 함께하며 발생하는 문제를 해결합니다.

Tuist는 개발자를 위한 주요 진입점인 [**CLI와**](https://github.com/tuist/tuist) CLI가 상태를 유지하고 다른 공개 서비스와 통합하기 위해 통합되는 서버로 구성되어 있습니다. 서버에 필요한 기능은 구독이 필요할 수 있습니다.

## 왜 Tuist를 사용해야 하나요?

왜 Tuist를 선택해야 할까요? 다음과 같은 강력한 이유가 있습니다:

- **모듈화 간소화:** 프로젝트가 성장하고 여러 플랫폼에 걸쳐 확장됨에 따라 모듈화가 중요해집니다. Tuist는 이러한 복잡성을 간소화하여 프로젝트 구조를 최적화하고 더 잘 이해할 수 있는 도구를 제공합니다.
- **워크플로우 최적화**: Tuist는 프로젝트 정보를 활용하여 선택적 테스트 실행과 빌드 간 결정론적 바이너리 재사용을 통해 효율성을 향상시킵니다.
- **건강한 프로젝트 진화 촉진**: 프로젝트의 역학 관계에 대한 인사이트와 정보에 입각한 의사 결정을 위한 전문가 가이드를 제공합니다. 이러한 접근 방식은 개발자의 이탈과 비즈니스 목표 누락으로 이어질 수 있는 건강하지 않은 프로젝트와 관련된 좌절과 생산성 손실을 방지합니다.
- **비용이 많이 드는 플랫폼 팀을 교체하세요**: 비용이 많이 들고 잠재적으로 위험할 수 있는 사내 플랫폼 팀에 투자하는 대신 Tuist를 가상 전문가로 활용하세요. 핵심 개인에게 의존하는 취약점 없이 일관된 지원을 제공합니다.
- **사일로 해체**: 플랫폼별 에코시스템(예: Xcode의 폐쇄적인 환경)과 달리 Tuist는 웹 중심 환경을 제공하며 Slack, Prometheus, GitHub와 같은 인기 도구와 원활하게 통합되어 도구 간 협업을 강화합니다.

Tuist와 프로젝트, 회사에 대해 더 자세히 알고 싶으시다면 당사의 비전, 가치, 팀에 대한 자세한 정보가 담긴 [핸드북](https://handbook.tuist.io/) 을 확인해보세요.