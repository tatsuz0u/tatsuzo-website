---
tag: Code
title: BitRemote - 個人開発しているアプリの技術解説
date: 2023-05-11T06:56:04-0000
description: BitRemote は、Apple 全プラットフォーム対応を目指す、ダウンロードタスクを遠隔管理するアプリです。フル SwiftUI + TCA で作られていて、現時点で iOS・iPadOS・macOS を対応しています。今回は、アプリを作るのに用いた技術について解説していきたいと思います。
---

## はじめに
[**BitRemote**](https://bitremote.app) は、Apple 全プラットフォーム対応を目指す、ダウンロードタスクを遠隔管理するアプリです。フル SwiftUI + TCA で作られていて、現時点で iOS・iPadOS・macOS を対応しています。

今回は、アプリを作るのに用いた技術について解説していきたいと思います。

## マルチプラットフォーム
UI デザインが Apple-liked のため、SwiftUI の恩恵が受けられて、マルチプラットフォーム対応は低コストで実装できました。

1. UI コードはできるだけ再利用したいので、OS によっての出し分けは最低限に
2. 出し分けには Active Compilation Conditions と制御フロー（if else switch）を使う
    1. Active Compilation Conditions を使うメリット
        1. コードのトップレベルから使える
        2. else ケース内はビルドエラーにならないので、ターゲット OS では使えない API が書ける
        
        ```swift
        #if os(iOS)
        // iOS と iPadOS だけスワイプアクションの実装を入れる
        .swipeActions(content: {})
        #else
        // そのほかの OS はキーボードの削除キーが押されたら何か処理を行う
        // 今は macOS だけ、増えたらリファクタリング
        .onDeleteCommand(perform: {})
        #endif
        ```
        
    2. 制御フローを使うメリット
        1. 何も書かないとエラーになる Result Builder 内で else ケース書かなくてもいい（`if trueCondition { content }`）
        2. else ケース内でもビルドが通るコードしか書けないから、CI でターゲットが変わってビルドエラーになることはないから安心してリファクタリングできる
3. OS 専用の画面は別モジュールで管理している

「画面があらゆる OS でもちゃんと表示できるように」とかはあまり意識していなくて普段通りに実装していたんですが、それでもなんとなく良い感じに出来上がりました。🤔

## プレゼンぼかし
Stub は使っていなくていつも本番データでアプリの宣伝画像を作っているので、見せたくない情報を簡単に隠せるために実装しました。

```swift
public enum PresentationCenter {
    // ログの実装からこのプロパティを参照し、隠すべき情報がある場合ログの出力を止める
    public static var isRedated: Bool { !redatedScopes.isEmpty }
    #if DEBUG
    public static let redatedScopes: [PresentationRedatedScope] = .unlimited
    #else
    // 本番コードはいつもオフ（unlimited）にする
    public static let redatedScopes: [PresentationRedatedScope] = .unlimited
    #endif
}

private extension [PresentationRedatedScope] {
    static var unlimited: Self { .init() }
    static var lightweight: Self {
        [.clientUsername, .clientHost,.taskName]
    }
    static var standard: Self {
        [.clientUsername, .clientHost, .categoryName, .tagName, .taskName]
    }
    static var restricted: Self { Element.allCases }
}

public enum PresentationRedatedScope: CaseIterable {
    case clientName, clientUsername, clientHost, categoryName, tagName, taskName
}

public extension String {
    // `.navigationTitle` など View をカスタマイズできない箇所は値をいじる
    func redated(as scope: PresentationRedatedScope, when condition: Bool = true) -> String {
        PresentationCenter.redatedScopes.contains(scope) && condition ? L10n.Constant.presentationModeMark : self
    }
}

public extension View {
    func redated(as scope: PresentationRedatedScope, when condition: Bool = true) -> RedatedView<Self> {
        .init(content: self, scope: scope, condition: condition)
    }
}

public struct RedatedView<Content: View>: View {
    let content: Content
    let scope: PresentationRedatedScope
    let condition: Bool

    public var body: some View {
        // `redacted(:_)` API は条件を見ないので
        // こちらの実装は一種の Conditional Modifier
        // アンチパターンではあるんですけど、あまり条件を変えない場合は OK
        if PresentationCenter.redatedScopes.contains(scope) && condition {
            content.redacted(reason: .placeholder)
        } else {
            content
        }
    }
}
```

## データフロー
下記がデータフローの一部簡略化したものです。

<img alt="data-flow" width="800" src="/assets/img/posts/2023-05-11-bitremote-insight/data-flow.png">

## macOS アプリ開発
### キーボード
フォーカスされた画面内のボタンに `.keyboardShortcut` を付けると、定義したショートカット（例えば ⌘ + R）が押された場合、そのボタンのアクションが実行されます。

※: ただし、画面内というのは View Hierarchy に存在することで、`.contextMenu` 内のボタンは表示されるまで View Hierarchy にないのでショートカットが押されても反応しないです。

ショートカット定義以外にもキーボード機能を利用できる API があります。

- `.onExitCommand`（esc）
- `.onDeleteCommand`（⌫⌦）
- `.onMoveCommand`（←↑↓→）

（下記 API を利用すると、該当のメニュー項目も有効化されます。）

- `.onCutCommand` （⌘ + X）
- `.onCopyCommand` （⌘ + C）
- `.onPasteCommand` （⌘ + V）

### メニューバー
- `.commands` メニュー項目の追加・差し替え
- `MenuBarExtra` メニューバーのアプリアイコン表示
- `Settings` 「設定」というメニュー項目の遷移先画面
  - `.commands` でも同じく実装できるはずだが、多分 SwiftUI のラッパー API

### アクティブ状態
環境変数 `controlActiveState` で Control のアクティブ状態がわかるので、それによって外見を変えるように実装したらより Apple っぽくなれます。

※: Control とは、Button、Toggle、Slider などの UI コンポーネントのことを指します。

### ウィンドウ・カラムサイズ
- `.defaultSize` ウィンドウのデフォルトサイズ
  - 指定できるのはあくまで初期状態のサイズで、ユーザが変更したらそのサイズは OS にキャッシュされます
- `.navigationSplitViewColumnWidth` View の所属カラムの横幅

## 終わりに
> [**BitRemote**](https://bitremote.app) は現在、[App Store](https://apps.apple.com/ja/app/bitremote/id6477765303) にて公開中です。
> 
> よかったらアプリを触っていただけると嬉しいです！

最後までお読みいただき、ありがとうございました！