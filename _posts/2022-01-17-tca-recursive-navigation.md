---
tag: Code
title: TCA で Recursive Navigation 実装してみた
date: 2022-01-17T03:38:34-0000
description: 本記事では、Recursive Navigation （再帰的ナビゲーション）は `View A -> View B -> View A -> View B -> …` みたいに延々とナビゲートしていくパターンを指します。それを TCA でどうやって実装するのかについて解説したいと思います。🙇‍♂️
---

## はじめに
> TCA (The Composable Architecture)
> 
> [pointfreeco/swift-composable-architecture](https://github.com/pointfreeco/swift-composable-architecture)

本記事では、Recursive Navigation （再帰的ナビゲーション）は

`View A -> View B -> View A -> View B -> …`

みたいに延々とナビゲートしていくパターンを指します。

これをどうやって TCA で実装するのかについて解説したいと思います。🙇‍♂️

## 簡単なところから
```swift
init(@ViewBuilder destination: () -> Destination, @ViewBuilder label: () -> Label)
```

この `NavigationLink` イニシャライザを使えば、`isActive` や `selection` を指定せずに、SwiftUI に任せることで簡単に作れます。僕のプロジェクトでも TCA が導入された前は使っていました。

しかし、その簡単さゆえに、できることも限られます。ナビゲーション情報がないから、ユニットテストや DeepLink はまず無理でしょう。🙅‍♂️

## とりあえずの試み
ではナビゲーション情報を state に保存してみましょう。

TCA で最も汎用的な実装ですが、
```swift
struct AaaState: Equatable {
  enum Route {
    case bbb
  }

  @BindableState var route: Route?
  var bbbState = BbbState()
}

enum AaaAction: BindableAction {
  case binding(BindingAction<AaaState>)
  case setNavigation(AaaState.Route?)
  case bbb(BbbAction)
}

struct AaaEnvironment {}

let aaaReducer = Reducer<AaaState, AaaAction, AaaEnvironment>.combine(
  .init { state, action, _ in
    switch action {
    case .binding:
      return .none
    case .setNavigation(let route):
      state.route = route
      return .none
    case .bbb:
      return .none
    }
  }
  .binding(),
  bbbReducer.pullback(
    state: \.bbbState,
    action: /AaaAction.bbb,
    environment: { _ in .init() }
  )
)

// BbbState, BbbAction, BbbEnvironment, bbbReducer は空っぽです
```

これで View A の方で `viewStore.send(.setNavigation(.bbb))` を使えば View B にナビゲートされて、`View A -> View B` 片方のルートは確保完了だと思います。

それで `BbbState`、`BbbAction`  や `bbbReducer` にも Aaa を入れてみますと、

- ❌ Circular reference
- ❌ Recursive enum ‘BbbAction’ is not marked ‘indirect’
- ❌ Value type ‘BbbState’ cannot have a stored property that recursively contains it

コンパイラにめっちゃ怒られました。ですよね…入れちゃまずいですよね…

`BbbAction` に `indirect` マークを入れれば二つ目のエラーは解消されるんですけど、他の二つはやはりこの構造だと無理でしょう。

## 辿り着いたかも
ググって最初に見つけたのは TCA 公式の CaseStudies でした。

[HigherOrderReducers-Recursion](https://github.com/pointfreeco/swift-composable-architecture/blob/main/Examples/CaseStudies/SwiftUICaseStudies/04-HigherOrderReducers-Recursion.swift)

なるほど、最初からお互いを含める構造ではなければ大丈夫ですね。

では `BbbState`  と `BbbAction` を少し変更してみますと、

```swift
struct BbbState: Equatable {
  // ...
  var aaaStates = IdentifiedArrayOf<AaaState>()
}

enum BbbAction: BindableAction {
  // ...
  indirect case aaa(id: String, action: AaaAction)
}
```

`AaaState` の方も `Identifiable` に conform して、

```swift
struct AaaState: Equatable, Identifiable {
  // ...
  let id: String
  init(id: String = UUID().uuidString) {
    self.id = id
  }
}
```

ひとまずコンパイラ鎮めることはできました。

それで `bbbReducer` にも `aaaReducer` 入れますと、

```swift
let bbbReducer = Reducer<BbbState, BbbAction, BbbEnvironment>.combine(
  .init { state, action, _ in
    // ...
  }
  .binding(),
  aaaReducer.forEach(
    state: \BbbState.aaaStates,
    action: /BbbAction.aaa(id:action:),
    environment: { _ in AaaEnvironment() }
  )
)
```

ひとまずコンパイラは鎮め、てない？

- ❌ Circular reference

またお前か…🤨

reducer たちがお互いを含めていて、また無限にループしますのでダメです。

では、reducer も Optional 型にしてみましょうか。

```swift
struct BbbState: Equatable {
  // ...

  // Reducer は Equatable ではないので
  static func == (lhs: BbbState, rhs: BbbState) -> Bool {
    lhs.route == rhs.route
    && lhs.aaaStates == rhs.aaaStates
    && (lhs.aaaReducer == nil) == (rhs.aaaReducer == nil)
  }
  var aaaReducer: Reducer<AaaState, AaaAction, AaaEnvironment>?
}

let bbbReducer = Reducer<BbbState, BbbAction, BbbEnvironment> { state, action, environment in
  switch action {
  // ...
  case .aaa:
    guard let aaaReducer = state.aaaReducer else { return .none }
    return aaaReducer.forEach(
      state: \BbbState.aaaStates,
      action: /BbbAction.aaa(id:action:),
      environment: { (environment: BbbEnvironment) in .init() }
    )
    .run(&state, action, environment)
  }
}
.binding()
```

こうやって reducer が何か値がある場合だけ action を処理できるようにすれば、循環参照エラーは解消されます。

値を与えるための action も入れてみましょう。

```swift
enum BbbAction: BindableAction {
  // ...
  case initializeRecursiveStates(String)
}

// 直接書くとやはりダメらしいです
// この workaround で特に問題無さそう
var anyAaaReducer: Reducer<AaaState, AaaAction, AaaEnvironment> {
  aaaReducer
}
let bbbReducer = Reducer<BbbState, BbbAction, BbbEnvironment> { state, action, environment in
  switch action {
  // ...
  case .initializeRecursiveStates(let identifier):
    state.aaaReducer = anyAaaReducer
    state.aaaStates = .init(uniqueElements: [.init(id: identifier)])
    return .none
  }
}
```

これで `setNavigation` を使う前に `initializeRecursiveStates` を使えば、`View B -> View A` のナビゲーションも可能になります。実装完了です。🎉

## クラッシュ対応策
本来上記の「実装完了です」のところで終わりましたが、そのあと実機でこの実装のあるところにいくとクラッシュすることがわかりましたので、対応策についても解説したいと思います。

クラッシュの原因は構造の複雑さによるスタックオーバーフローだと思います。下記リンクのように、スタックに負担の大きい state をヒープに保存させれば、クラッシュはある程度抑えられるでしょう。
[Potential for stack overflow with large app states](https://github.com/pointfreeco/swift-composable-architecture/discussions/488)

まずはプロパティをヒープに保存させるためのラッパーを用意しましょう。
```swift
private final class Reference<T: Equatable>: Equatable {
  var value: T
  init(_ value: T) {
    self.value = value
  }
  static func == (lhs: Reference<T>, rhs: Reference<T>) -> Bool {
    lhs.value == rhs.value
  }
}

@propertyWrapper struct Heap<T: Equatable>: Equatable {
  private var reference: Reference<T>

  init(_ value: T) {
    reference = .init(value)
  }

  var wrappedValue: T {
    get { reference.value }
    set {
      if !isKnownUniquelyReferenced(&reference) {
        reference = .init(newValue)
        return
      }
      reference.value = newValue
    }
  }
  var projectedValue: Heap<T> {
    self
  }
}
```

ラッパーの使い方は下記の感じです。
```swift
struct AaaState: Equatable {
  // ...
  init() {
    _bbbState = .init(nil)
    _cccState = .init(.init())
  }

  @Heap var bbbState: BbbState?
  @Heap var cccState: CccState!
}
```
循環参照にならないように必要に応じて一般な Optional 型（値の代入を忘れずに）か暗黙的アンラップ型を使ってください。

`Identifiable` の conform と `IdentifiedArray` はもう不要です。

次は [recurse-navigation-tco](https://github.com/ChaseStas/recurse-navigation-tco) のようにして、Optional 型の reducer プロパティも捨てて == メソッドを自分で書かなくて済むようにしましょう。

```swift
static func recurse(_ reducer: @escaping (Reducer) -> Reducer) -> Reducer {
  var `self`: Reducer!
  self = Reducer { state, action, environment in
    reducer(self).run(&state, action, environment)
  }
  return self
}

let aaaReducer = Reducer<AaaState, AaaAction, AaaEnvironment>.recurse { (self) in
  .combine(
    .init { state, action, environment in
      switch action {
      // ...
      case .bbb(.aaa(let recursiveAction)):
        guard state.bbbState != nil else { return .none }
        return self.run(&state.bbbState!.aaaState, recursiveAction, environment)
          .map({ AaaAction.bbb(.aaa($0)) })
      case .bbb:
        return .none
      }
    }
    .binding(),
    bbbReducer.optional().pullback(
      state: \.bbbState,
      action: /AaaAction.bbb,
      environment: { _ in .init() }
    )
  )
}
```
要は次回ループの自分の action の reduce 方法を予め定義して、reducer たちがお互いを含めないようにするんですね。🤔

リンク先の実装方法では `bbbReducer` の方も `recursiveAction` の対応も必要ですけど、使ってみたところなくても問題なさそうです。

ただし、ナビゲーションの順番を間違えると action が全く処理してもらえないらしいです。（`bbbReducer` で上記の対応をしても発生するらしい）

全く別物になりましたが、これでクラッシュ対応策の実装も完了です。🎉

テストしたところ、ループ 15 回以内は大丈夫そうです。

## 終わりに
> [**EhPanda**](https://ehpanda.app) というプロジェクトの開発・保守をやっています。
> 
> 本記事の実装も含まれていますので、興味がある方はぜひご覧になってください。
> 
> （`DetailView` が View A で `CommentsView` や `DetailSearchView` が View B です）

最後までお読みいただき、ありがとうございました！