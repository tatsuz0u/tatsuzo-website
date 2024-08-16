---
tag: Code
title: Live Activities & Dynamic Island
date: 2022-10-25T15:28:08-0000
---

## はじめに
Live Activities 機能やそれを実装する API について解説していきたいと思います。Dynamic Island は機能というより表示領域みたいなもんなんですが、目を引くのでタイトルに入れました。XD


### 用語説明
- ActivityKit：API 名
- Live Activities：機能名
- Live Activity：ActivityKit を用いて作られた単体

### 関連ページ
- [Live Activities (HIG)](https://developer.apple.com/design/human-interface-guidelines/components/system-experiences/live-activities)
- [ActivityKit (Apple Documentation)](https://developer.apple.com/documentation/ActivityKit)

## Live Activities とは
- フォアグラウンドでなくてもユーザが関心を持つような、最も「今」を表すアプリの状態を OS が提供した箇所で表示する機能
- OS の縄張りなので、請求・更新・終了を提案することができるけど、OS 側はそれを OS の都合によって表示してくれるだけ
- Dynamic Island とロック画面で同時に表示される
	- Dynamic Island がない機種ではロック画面のみ
- Live Activities を表示させるには
	- ActivityKit のみ（ローカル）
	- ActivityKit + APNS（リモート通知）

### アプリ内通知で使えるのか
- 結論は多分できない
- ローカル検証済み
	- バックグラウンドに入らないと Dynamic Island から出てこない
- リモート通知はまだ検証中
- 自前実装も悪くない、しかも全機種対応可能
	- [Dynamic Island liked notification concept (Twitter)](https://twitter.com/_Kavsoft/status/1576194926949568513)
	- [Dynamic Island - In-App Custom Notification's (YouTube)](https://www.youtube.com/watch?v=RhZwQj9D80c)


### Widget との関係
- Live Activities の UI 実装はアプリの Widget Extension の中に書く
	- 既存 Widget と一緒でもいいし、Live Activities のみでもいい
- Widget とは似て非なり、Widget のようにネットワーク通信や位置情報更新の受信ができない


## Dynamic Island レイアウト


### コンパクト
Live Activities に Activity が一個しかない場合にはこのサイズで表示される

<img alt="dynamic-island-compact" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-compact.png">

### ミニマム
複数の Live Activities が存在する場合、OS 側のプライオリティ基準に沿って、その中の一個を先端に置いて Dynamic Island にアタッチしたレイアウトで表示してくれる

<img alt="dynamic-island-minimal" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-minimal.png">

### 拡大
ユーザが長押しすると表示される

<img alt="dynamic-island-expanded" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-expanded.png">

### 実装について
- 他のアプリも Live Activity を出してるみたいな原因で、必ずしも Dynamic Island のコンパクトで表示できる訳ではない
- 都合でコンパクトで表示されない場合でも、ユーザがインタラクトしてくれたら拡大サイズで表示されるので、コンパクト・ミニマム・拡大サイズは必ず全部用意しましょう（しないと多分リジェクトされる）

### ちょっとハードウェアの話も
- Dynamic Island はスクリーンではない部分もタッチできる
- カメラ・環境光センサー・近接センサーでタッチを検知して（推測）、スクリーンをタッチしたような動きをしてくれる

<img alt="dynamic-island-hardware" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-hardware.jpg">

## API 挙動
- アプリ・ユーザが終了にしない限り、Live Activity は最長 8 時間アクティブでいられる
- アクティブでなくなると、OS に Dynamic Island から外される、ロック画面では追加で 4 時間留まることが可能（ユーザがそれを外さない限り）
- 結論として、アプリ・ユーザが何もしない場合、一個の Live Activity は Dynamic Island に 8 時間、ロック画面に 12 時間留まることが可能

## 実装いじってみてわかったこと
- 中心領域が求める幅をまず満足させてから、残りを先端・末端領域に分ける

<img alt="dynamic-island-hardware" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-implement-1.jpg">

- 既定値では、先端・末端領域は 50:50 で分けるが、プライオリティは指定可能
  - 中心領域に低いプライオリティを指定しても無視されるらしい

<img alt="dynamic-island-hardware" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-implement-2.jpg">

<img alt="dynamic-island-hardware" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-implement-3.jpg">

- 下部領域は幅に関わらず、中心領域ではなく先端・末端領域と隣接する

<img alt="dynamic-island-hardware" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-implement-4.jpg">


- Dynamic Island からレイアウトがはみ出てしまうとクリップされる

<img alt="dynamic-island-hardware" width="800" src="/assets/img/posts/2022-10-26-live-activities-and-dynamic-island/dynamic-island-implement-5.jpg">

## 終わりに
最後までお読みいただき、ありがとうございました！