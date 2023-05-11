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

## Table カラムのカスタマイズ
**⚠️: こちらの内容は、一部誤りがあるかもしれないです。**

`TableColumnBuilder` API 制限

- 制御フローが書けない
- `ForEach` が書けない

結局手入力で指定できる内容しか書けなくて、Apple はデベロッパにカラムを動的に変えてほしくないようです。

```swift
Table(
    data,
    columns: {
        nameColumn
        sizeColumn
        // ...
    }
)
```

そこで考えたのは、Table ごと可能なケースを全部用意すればできるじゃないですか。🤔

ただし、可能なケースというのは、実装したい機能によって変わります。

### 組み合わせと順列
カラムを非表示にするだけなら、それは項目数可変・項目順番不変の組み合わせ（Combinations）です。しかし、カラムの順番も指定したい場合は、項目順番も可変なので順列（Permutations）になります。

```swift
let columns = [1, 2, 3]

// Combinations
// `1...` == 空き配列を除いて全てのケース
print(columns.combinations(ofCount: 1...))
// Result (7 elements): [[1], [2], [3], [1, 2], [1, 3], [2, 3], [1, 2, 3]]

// Permutations
print(columns.permutations(ofCount: 1...))
// Result (15 elements): [[1], [2], [3], [1, 2], [1, 3], **[2, 1]**, [2, 3], **[3, 1]**,
// **[3, 2]**, [1, 2, 3], **[1, 3, 2]**, **[2, 1, 3]**, **[2, 3, 1]**, **[3, 1, 2]**, **[3, 2, 1]**]
```

ご覧の通り、カラム数がたったの 3 つでも、順列のケース数は 2 倍以上に増えます。

では僕のアプリのように、カラム数が 9 つある場合はどうなるでしょうか？
    
| 機能 / 数量 | 非表示のみ | 非表示 + 順番指定 |
| --- | --- | --- |
| ケース数 | 511 | 986,409 |
| ファイル数（100 行） | 6 | 9,865 |
| ファイル数（300 行） | 2 | 3,289 |

組み合わせの 1,930 倍ほど、およそ百万ケース・一万ファイルに劇的に膨大化します。

ファイル内のコードはほとんど UI コードなので、行数が多すぎるとコンパイラが時間かかりすぎてビルド失敗します。ちなみに、100 行のコード量はこんな感じです。

<img alt="generated-code" width="800" src="/assets/img/posts/2023-05-11-bitremote-insight/generated-code.png">

こんなファイルを一万個近く用意しないと順番指定機能が実装できないので、ここまでなれば、コンパイラの事情も考えるとすべてのケースをアプリに詰め込むのはもはや不可能でしょう。なので最後は非表示機能のみ対応することにしました。

### 改善案
やはり最初になんとなく勘づいた通り、百万近くのケースを用意しないといけないのは考えすぎでした。

`TableColumnBuilder` API に制限があるとはいえ、`buildBlock` は十個まで用意してくれているので、制御フローや `ForEach` を使わずに羅列してあげれば大丈夫なはずです。

```swift
switch columnTypes.count {
// ...
case let value where value >= 10:
    Table(
        rows,
        selection: selection,
        sortOrder: sortOrder,
        columns: { 
            columnTypes[0].column
            columnTypes[1].column
            columnTypes[2].column
            columnTypes[3].column
            columnTypes[4].column
            columnTypes[5].column
            columnTypes[6].column
            columnTypes[7].column
            columnTypes[8].column
            columnTypes[9].column
        }
    )
}
```

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
> [**BitRemote**](https://bitremote.app) は現在、TestFlight にて公開テスト中です。
> 
> 枠は十分にあると思いますので、よかったらアプリを触っていただけると嬉しいです！

最後までお読みいただき、ありがとうございました！