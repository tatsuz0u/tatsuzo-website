---
tag: Code
title: GitHub Actions 建構 iOS CI/CD 自動化流程
date: 2021-09-21 05:42:13-0000
---

我目前正在維護 [**EhPanda**](https://ehpanda.app) 這個開源專案，在引入 GitHub Actions 之前，我每次發佈新版本都要手動執行 「測試、更新版本、打包、發佈到 GitHub & ASC & AltStore、發佈更新日誌」 這一冗長流程，終於在朋友的念念叨叨和我本人積壓的不滿下，我覺得該作出改變了...

> 本文預設讀者已經瞭解 GitHub Actions 的定義和用法，如果你的情況並非如此，可參考以下連結。  
>  [GitHub Actions Documentation](https://docs.github.com/en/actions)

## 準備工作
Van 事開頭難，我看完官方文件之後，對著語法說明，嗯...沒有頭緒。下一步，Google 「GitHub Actions iOS CI/CD」，想先找一篇範例看看。現在用這個關鍵字可能會搜到本文🌚，總之我決定先把這一段丟進我的草稿。

```yml
name: Build
on: [push]
env:
  DEVELOPER_DIR: /Applications/Xcode_13.0.app
jobs:
  Build:
    runs-on: macos-11
    steps:
      - name: Checkout
        uses: actions/checkout@v2
```

* `DEVELOPER_DIR`  或  `xcode-select`  指定 Xcode 版本
* `actions/checkout`  取得 repo 內容

> 可透過以下連結查詢 GitHub Actions 支援的作業系統及 Xcode 版本。  
> [About GitHub-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners)  

建立一個 yml 檔案，將內容寫入，並將檔案放在 GitHub repo 的  `.github/workflows`  資料夾下，GitHub 就能找到它了。

## 測試
先挑軟柿子捏，我想要每次提交程式碼後可以幫我跑一下測試。
```yml
- name: Run tests
  run: xcodebuild clean test
    -scheme <scheme_name> -sdk iphonesimulator
    -destination 'platform=iOS Simulator,name=iPhone 13'
```

- 可以在 env 定義或在此填寫明文  `<scheme_name>`
- 定義在 env 的變數可透過  `{% raw %}${{ env.<variable_name> }}{% endraw %}`  取得
- 一些專案使用  `xcodebuild`  可能需要指定  `-project`  或  `-workspace`

## 稍作調整
看起來我的 CI 需求就已經滿足了🤔，下面來看一下 CD。
建立一個 yml 檔案，跟剛才的區分開。
```yml
name: Deploy
on:
  pull_request:
    branches:
      - main
    types: [closed]
env:
    DEVELOPER_DIR: /Applications/Xcode_13.0.app
jobs:
  Deploy:
    runs-on: macos-11
    if: github.event.pull_request.merged == true && github.event.pull_request.user.login == '<user_name>'
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Run tests
        run: xcodebuild clean test 
          -scheme <scheme_name> -sdk iphonesimulator
          -destination 'platform=iOS Simulator,name=iPhone 13'
```

- `<scheme_name>`  和  `<user_name>`  處填寫自己的資訊喔
- 調整 Action 的觸發條件為「一個目標分支為  `main` 、發起人的用戶名為  `<user_name>` 的 Pull Reqeust 被關閉、被合併」

> 可透過以下連結查看更多 GitHub Event 相關資訊。  
> [Events that trigger workflows](https://docs.github.com/en/actions/reference/events-that-trigger-workflows)  
> [Webhook events and payloads](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads)  

## 證書和供應配置文件

進行解碼、安裝工作前，先確認需要哪些檔案。
- 上傳至 ASC: ASC API 密鑰
- 封存: 開發環境的證書、供應配置文件
- IPA 匯出: 生產環境的證書、供應配置文件

### 存放 GitHub Secrets
`base64 -i <file_path> | pbcopy`  將需要的檔案編碼為 base64 並拷貝至剪貼簿。從 repo 的  `Settings - Secrets`  找到 Actions secrets，點擊右上角的  `New repository secrets`  ，填寫 secret 名稱和內容並確認添加。

### 解碼
在 GitHub Action 中可以透過  `{% raw %}${{ secrets.<secret_name> }}{% endraw %}`  的形式取得存放在 GitHub Secrets 的內容。

在 env 添加路徑變數，以下內容僅供參考。
```yml
env:
    ...
    BUILDS_PATH: '/tmp/action-builds'
    ASC_KEY_PATH: '/Users/runner/private_keys'
```

在 env 定義好變數之後就開始解碼。
```
- name: Decode certificates & provisioning profiles
    run: |
      mkdir <folder_path>
      echo -n <secrets_base64> | base64 -d -o <output_path>
      ...
```

- `...`  代表有省略前文或重複格式內容，如為後者可按需要自行補全

總之就是建立好資料夾之後  `echo -n <base64_content> | base64 -d -o  <output_file_path>`  將 GitHub Secrets 中存放的 base64 解碼。

### 安裝證書
```yml
- name: Install certificates
    run: |
      KEY_CHAIN=<keychain_name>.keychain-db
      P12_PASSWORD=<secrets_p12_password>
      PASSWORD='<keychain_password>'

      security create-keychain -p $PASSWORD $KEY_CHAIN
      security default-keychain -s $KEY_CHAIN
      security unlock-keychain -p $PASSWORD $KEY_CHAIN
      security set-keychain-settings -t 3600 -u $KEY_CHAIN

      security import <import_path> -k $KEY_CHAIN -T /usr/bin/codesign
      ...

      security set-key-partition-list -S apple-tool:,apple: -s -k $PASSWORD ~/Library/Keychains/$KEY_CHAIN
```

- `security create-keychain -p <keychian_password> <keychian_name>`  建立鑰匙圈
- `security default-keychain -s <keychian_name>`  設定為預設鑰匙圈
- `security unlock-keychain -p <keychian_password> <keychian_name>`  解鎖鑰匙圈
- `security set-keychain-settings -t 3600 -u <keychian_name>`  設定鑰匙圈在 3600 秒後鎖定
- `security import $PATH -k <keychian_name> -T /usr/bin/codesign`  匯入證書及密鑰
- `security set-key-partition-list -S apple-tool:,apple: -s -k  <keychain_password> <keychian_path>`  應該是用來避免出現 GUI 的

### 安裝供應配置文件
```yml
- name: Install provisioning profiles
    run: |
      mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
      uuid=`grep UUID -A1 -a <provisioning_profile_path> | grep -io "[-A-F0-9]\{36\}"`
      cp <provisioning_profile_path> ~/Library/MobileDevice/Provisioning\ Profiles/$uuid.mobileprovision
      ...
```

## 更新版本
```yml
- name: Bump version
    id: bump-version
    uses: yanamura/ios-bump-version@v1
    with:
      version: <app_version>
```

- 在這裡定義  `app_version`  ，不需要在 Xcode 管理了喔
- [yanamura/ios-bump-version](https://github.com/yanamura/ios-bump-version)，它透過  `agvtool`  更新，我將它稍作修改使它可以輸出更新後的版本和 build，因此需要定義該 step 的 id，稍後需要透過這個 id 取得輸出結果
- 如果各 Target 的 `Info.plist`  中  `CFBundleVersion`  不存在，可以手動添加；如果  `project.pbxproj`  中的 `CURRENT_PROJECT_VERSION`  在更新後對應不上最新的 build 號，可以將其移除

## 打包

### 封存
```yml
- name: Xcode archive
    run: xcodebuild archive 
      -scheme <scheme_name>
      -destination 'generic/platform=iOS'
      -archivePath <archive_path>
```

### 匯出 IPA
```yml
- name: Export .ipa file
    run: xcodebuild -exportArchive
      -exportPath <export_path> 
      -archivePath <archive_path>
      -exportOptionsPlist <plist_path>
```

- 與封存的  `-archivePath`  不同，`xcarchive`  本身是一個資料夾就沒差，但是  `-exportPath`  要指定的是匯出結果的資料夾路徑，如果寫一個  `/path/to/ipa/xxx.ipa`  這樣的路徑，它會把  `xxx.ipa`  視為資料夾並在裡面存放匯出的結果
- 提前把  `ExportOptions.plist`  放到方便取得的位置，需要模板的話可以用 Xcode 手動執行匯出，匯出資料夾裡面就會有

## 發佈

### 設定 Git 作者
後面會遇到需要 commit 的地方，先設定好 Git 作者。
在 steps 的最開始，取得 repo 內容的 Checkout action 後面加上。
```yml
...
- name: Modify git config
    run: |
      git config user.name "github-actions[bot]"
      git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
...
```

- 建議使用這個 name 和 email，可以在 commits 處看到符合預期的頭貼

### 取得並檢查資料
```yml
- name: Retrieve data
    id: retrieve-data
    run: |
      echo "::set-output name=<output_name>::$(<output_content>)"
      ...

- name: Validate data
    run: |
      [[ ! -z "{% raw %}${{ steps.<step_id>.outputs.<output_name> }}{% endraw %}" ]] || exit 1
      ...
```

- `stat -f%z <file_path>`  可取得 ipa 檔案大小
- `date -u +"%Y-%m-%dT%T"`  可取得當前 UTC + 0 時間
- `[[ ! -z "<content>" ]] || exit 1`  判斷內容為空時退出，起到檢查資料的作用，可以在發佈前結束工作流減少損失

### 發佈到 GitHub
```yml
- name: Release to GitHub
    uses: softprops/action-gh-release@v1
    with:
      fail_on_unmatched_files: true
      token: <github_personal_access_token>
      ...
```

> 可透過以下連結查看更多該 Action 用法。  
> [softprops/action-gh-release](https://github.com/softprops/action-gh-release)  

### 發佈到 App Store Connect
```yml
- name: Upload to ASC
    run: xcrun altool --upload-app -t ios
      -f {% raw %}${{ env.IPA_OUTPUT_PATH }}{% endraw %}
      --apiKey <secrets_asc_key_id>
      --apiIssuer <secrets_asc_issuer_id>
```

- `altool`  也可以使用  `-u <apple_id> -p <app_password>`  的方式，缺點是可能遇到雙重認證，需要用這種方式可以到 [Manage your Apple ID](https://appleid.apple.com/) 生成 App 專用密碼
- `--apiKey <key_id> --apiIssuer <issuer_id>`  的方式所需的資料可以在 [App Store Connect](https://appstoreconnect.apple.com) 的  `Users and Access - Keys`  取得

### 發佈到 AltStore
在發佈之前，需要先提交更新版本的更改。
```yml
- name: Commit bump version
    run: |
      git add .
      git commit -m "Bump version"
      git push origin HEAD
```

AltStore 是以添加源的形式運作的，因此只需更新源的 json 就完成了發佈。
```yml
- name: Update AltStore.json
    run: |
      echo "`jq '.<array>[<index>].<key>=<value>' <from_path>`" > <to_path>
      ...

- name: Commit update AltStore.json
    run: |
      git add .
      git commit -m "Update AltStore.json"
      git push origin HEAD
```

## 發佈更新日誌
> 可透過以下連結查看更多 API 用法。  
> [Telegram Bot API](https://core.telegram.org/bots/api)  
> [Discord Developer Portal](https://discord.com/developers/docs/resources/webhook)  
> [curl - Discord Webhooks Guide](https://birdie0.github.io/discord-webhooks-guide/tools/curl.html)  

### 發佈到 Telegram
聯繫 [@BotFather](https://t.me/BotFather) ，建立一個 bot 並取得它的認證 token。讓你的 bot 加入需要發佈更新日誌的群組或頻道，透過  `https://api.telegram.org/bot<token>/getUpdates`  API 取得 Channel ID。

```yml
- name: Post release notes
  run: |
    curl https://api.telegram.org/bot<token>/sendMessage \
    -d parse_mode=markdown -d chat_id=<channel_id> \
    -d text='<message_content>'
```

### 發佈到 Discord
`Edit Channel - Integrations - Webhooks - Create Webhook`  建立  Webhook 並取得 Webhook URL 。

在 Telegram 的後面加上。
```yml
    ...
    curl <secrets_webhook_url> \
    -F 'payload_json={"content": "<message_content>"}'
```yml

## 結束語
這個工作流按我的需求到這裡就結束了，實作細節可參考以下連結。  
 [EhPanda/.github/workflows](https://github.com/tatsuz0u/EhPanda-Team/tree/main/.github/workflows)