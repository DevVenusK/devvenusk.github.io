---
layout: post
title: [Tuist] Install Tuist
date: "2024-09-30 00:00:00 +0900"
categories: ["Tuist"]
tags:
- tuist
- iOS
- guide
type: post
published: true
meta: {}
---
Tuist CLI는 실행 가능한 동적 프레임워크와 일련의 리소스(예: 템플릿)로 구성되어 있습니다. [소스에서](“https://github.com/tuist/tuist”) 수동으로 Tuist를 빌드할 수도 있지만, **올바른 설치를 위해 다음 설치 방법 중 하나를 사용하는 것이 좋습니다.**

## 권장: [Mise](https://github.com/jdx/mise)
Tuist의 버전을 결정적으로 관리하고 활성화하기 위한 도구로 [Mise](https://github.com/jdx/mise)가 기본값으로 제공됩니다. 시스템에 아직 설치되어 있지 않은 경우 다음 [설치 방법](https://mise.jdx.dev/getting-started.html) 중 하나를 사용할 수 있습니다. 터미널 세션에서 Tuist 프로젝트 디렉터리를 선택할 때 올바른 버전이 활성화되도록 셸에 제안된 줄을 추가하는 것을 잊지 마세요.

>INFO  
Mise는 디렉토리에 대한 버전 범위 지정 및 활성화를 지원하여 모든 환경에서 동일한 버전의 Tuist를 결정론적으로 사용할 수 있도록 하기 때문에 [HomeBrew](https://brew.sh)와 같은 대안보다 권장됩니다.

설치가 완료되면 다음 명령어 중 하나를 통해 Tuist를 설치할 수 있습니다:
```bash
mise install tuist # .tool-versions/.mise.toml에 지정된 현재 버전을 설치합니다.
mise install tuist@x.y.z # 특정 버전 번호 설치
mise install tuist@3 # 퍼지 버전 번호 설치
```
단일 버전의 도구를 전역적으로 설치 및 활성화하는 Homebrew와 같은 도구와 달리 **Mise는 전역적으로 또는 프로젝트에 한정된 버전을 활성화해야 한다는 점**에 유의하세요. 이 작업은 `mise use`를 실행하여 수행합니다:
```bash
mise use tuist@x.y.z # 현재 프로젝트에서 tuist-x.y.z 사용
mise use tuist@latest # 현재 디렉터리에서 최신 tuist를 사용합니다.
mise use -g tuist@x.y.z # 글로벌 기본값으로 tuist-x.y.z 사용
mise use -g tuist@system # 시스템의 tuist를 전역 기본값으로 사용합니다.
```
## 대안: [HomeBrew](https://brew.sh)
여러 환경 간에 버전 고정이 문제가 되지 않는다면 [Homebrew](https://brew.sh)와 [다음 공식](https://github.com/tuist/homebrew-tuist)을 사용하여 Tuist를 설치할 수 있습니다:**
```bash
brew tap tuist/tuist
brew install --formula tuist
brew install --formula tuist@x.y.z
```
## Shell 완성
Tuist를 **전역으로 설치**한 경우, 명령과 옵션을 자동 완성하는 Bash 및 Zsh용 셸 완성 기능을 설치할 수 있습니다.

>전역 설치란 무엇인가요?*
전역 설치는 셸의 $PATH 환경 변수에서 사용할 수 있는 설치입니다. 즉, 터미널의 모든 디렉토리에서 튜이스트를 실행할 수 있습니다.이것은 Homebrew의 기본 설치 방법입니다.

## Zsh
[oh-my-zsh](https://ohmyz.sh)가 설치되어 있다면 이미 완성 스크립트를 자동으로 로드하는 디렉터리인 `.oh-my-zsh/completions`가 있습니다. 새 완성 스크립트를 해당 디렉터리의 `_tuist`라는 새 파일에 복사합니다:
```bash
tuist --generate-completion-script > ~/.oh-my-zsh/completions/_tuist
```

[oh-my-zsh](https://ohmyz.sh)가 없는 경우 함수 경로에 완성 스크립트 경로를 추가하고 완성 스크립트 자동 로딩을 사용 설정해야 합니다. 먼저 `~/.zshrc`에 다음 줄을 추가합니다:
```bash
fpath=(~/.zsh/completion $fpath)
autoload -U compinit
compinit
```
그런 다음 `~/.zsh/complication`에 디렉터리를 만들고 완료 스크립트를 새 디렉터리에 다시 `_tuist`라는 파일에 복사합니다.
```bash
tuist --generate-completion-script > ~/.zsh/completion/_tuist
```
## Bash
[bash-complement](https://github.com/scop/bash-completion)가 설치되어 있는 경우 새 완료 스크립트를 `/usr/local/etc/bash_complement.d/_tuist` 파일에 복사하기만 하면 됩니다:
```bash
tuist --생성-완성 스크립트 > /usr/local/etc/bash_completement.d/_tuist
```
bash-completion이 없으면 완성 스크립트를 직접 소싱해야 합니다. `~/.bash_completions/`와 같은 디렉터리에 복사한 다음 `~/.bash_profile` 또는 `~/.bashrc`에 다음 줄을 추가합니다:
```bash
source ~/.bash_completions/example.bash
```
