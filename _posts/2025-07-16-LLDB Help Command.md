---
layout: post
title: LLDB Help Command
date: 2025-07-16 00:00:00 +0900
categories:
  - iOS/debug
tags:
  - iOS
  - debug
  - lldb
  - Xcode
type: post
published: true
meta: {}
---
`LLDB` 는 `help` 명령어를 제공한다.
```
(lldb) help
```
이 명령어는 사용할 수 있는 모든 명령어를 dump 하며, `~/.lldbinit` 에 존재하는 커스텀 명령어도 포함한다.

`LLDB` 는 `apropos` 명령어도 제공한다.
`apropos` 는 LLDB 문서에서 단어나 문자열에 대해 검색하여 일치하는 결과를 반환한다.
```
(lldb) apropos swift
(lldb) apropos "refrence count"
```
> NOTE:
> apropos 는 하나의 argument 만 가능하므로 "" 로 묶어주어야 하나의 argument  로 처리가 가능하다.