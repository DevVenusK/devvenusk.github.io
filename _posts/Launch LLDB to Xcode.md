---
created: 2024-08-05
layout: post
tags:
  - Blog
  - LLDB
share: "true"
---
1. 터미널을 열고 이름을 `LLDB`로 변경한다.
2. 터미널에서 `lldb`[^launchlldb]를 입력하여 `LLDB`를 실행한다.
3. 새로운 탭을 만들고 이름을 `Xcode stderr`로 변경한다.
4. 터미널에서 `tty`를 입력하면 아래와 유사한 주소가 나타난다: `/dev/ttys001`[^adressofterminal]
5. 터미널에서 `target create /Applications/Xcode-15.4.0.app/Contents/MacOS/Xcode`를 입력한다.
6. 터미널에서 `process launch -e /dev/ttys001 --`를 입력한다.

>NOTE:
>LLDB 는 디버깅 할 때 기본적으로 Objective-C Context 를 사용한다.

`Swift` 사용하기
```swift
(lldb) ex -l swift -- import Foundation 
(lldb) ex -l swift -- import AppKit
(lldb) expr -l swift -- import Cocoa
```

출처: [Advanced Apple Debugging Reverse Engineering](https://www.kodeco.com/books/advanced-apple-debugging-reverse-engineering/v4.0)

[^launchLLDB]: arm64체계에서는 `arch -arm64 lldb`
[^adressofterminal]: 터미널 주소