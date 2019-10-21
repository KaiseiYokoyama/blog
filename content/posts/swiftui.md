---
title: "SwiftUIでFor文を使う方法"
date: 2019-10-21T12:07:02+09:00
draft: false
tags: ["SwiftUI","Swift5","Swift","iOS", "Tips"]
---

題名をだいぶ省略しました。

```swift
struct HogeView: View {
  var body: some View {
    // ここ
  }
}
```

上記のコードのコメントの位置でFor文じみた文法を使用する方法です。

## TL;DR
SwiftUIの[**ForEach**](https://developer.apple.com/documentation/swiftui/foreach)を使います。

```swift
/// Int型の値を格納できるサンプル用の型
struct Value: Identifiable {
  var id = UUID()
  var value: Int
}

/// 子ビュー
struct FugaView: View {
  var body: some View {
    // 省略
  }

  init(_ value: Value) {
    // 省略
  }
}

/// 親ビュー
struct HogeView: View {
  var values = [Value(value: 0),Value(value: 1),Value(value: 2)]
  var body: some View {
    ForEach(values) { value in
      // 例: FugaViewをvaluesごとに生成する
      FugaView(value)
    }
  }
}
```

## 参考文献
- https://www.hackingwithswift.com/quick-start/swiftui/how-to-create-views-in-a-loop-using-foreach
