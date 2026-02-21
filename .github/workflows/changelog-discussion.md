---
description: GitHub Changelogの指定期間（日本時間）のエントリーを取得して、Discussionsにまとめます
on:
  workflow_dispatch:
    inputs:
      start_date:
        description: "集計開始日（JST: YYYY-MM-DD　例: 2026-02-16）"
        required: true
        type: string
      end_date:
        description: "集計終了日（JST: YYYY-MM-DD　例: 2026-02-28）"
        required: true
        type: string
      title:
        description: "Discussionタイトル（例: Radio 2026.02（後半））"
        required: true
        type: string
      category:
        description: "Discussionカテゴリslug（デフォルト: announcements）"
        required: false
        type: string
        default: "announcements"
permissions: read-all
tools:
  web-fetch:
network:
  allowed:
    - github.blog
safe-outputs:
  allowed-domains:
    - github.blog
  create-discussion:
    max: 1
---

# GitHub Changelog Discussion Creator

GitHub Changelog（https://github.blog/changelog/）から、指定された期間（日本時間）のエントリーを収集し、カテゴリ別にまとめたDiscussionを作成するエージェントです。

## 実行パラメーター

以下の入力値を使用してください:

- **集計開始日（JST）**: ${{ github.event.inputs.start_date }}
- **集計終了日（JST）**: ${{ github.event.inputs.end_date }}
- **Discussionタイトル**: ${{ github.event.inputs.title }}
- **カテゴリ**: ${{ github.event.inputs.category }}

## Step 1: GitHub Changelogの取得と日付フィルタリング

### フェッチするURL

`start_date` と `end_date` が含まれる年の changelog ページを取得してください。

- **URL形式**: `https://github.blog/changelog/YYYY/`（例: `https://github.blog/changelog/2026/`）
- 期間が複数年にまたがる場合は、関連するすべての年のページを取得してください

参考として RSSフィード `https://github.blog/changelog/feed/` も利用できます。

### 日付フィルタリングのルール

- 日本標準時（JST = UTC+9）で `start_date`（含む）から `end_date`（含む）の範囲内に公開されたエントリーを収集してください
- Changelogページに掲載されている日付（例: "FEB.20"）はUTCです
- 各エントリーのURLに含まれる日付（例: `2026-02-20`）をUTCとして解釈し、JSTの日付範囲と照合してください
  - UTC YYYY-MM-DD 00:00 = JST YYYY-MM-DD 09:00
  - JSTの `start_date` 00:00 = UTC 前日の 15:00 に相当
  - 境界ケースで迷った場合はエントリーを含める方向で処理してください

### データ収集

各エントリーについて以下を記録してください:
- **タイトル**（英語のまま）
- **URL**（例: `https://github.blog/changelog/2026-02-20-...`）
- **種類**: RELEASE / IMPROVEMENT / RETIRED のいずれか（ページ上のラベルから判断）
- **タグ**: COPILOT / ACTIONS / COLLABORATION TOOLS など

## Step 2: カテゴリ分け

収集したエントリーを以下のカテゴリに分類してください:

| カテゴリ名 | 対応するChangelogタグ・説明 |
|-----------|---------------------------|
| **Copilot** | COPILOTタグ（ただしAIモデル追加・廃止はModelsへ） |
| **Models** | AIモデルの新規提供・廃止・変更（OpenAI, Claude, Gemini等）が主題のエントリー |
| **Project & Issues** | PROJECTS & ISSUESタグ |
| **Collaboration tools & Community engagement** | COLLABORATION TOOLSタグ |
| **Actions** | ACTIONSタグ |
| **Codespaces** | CODESPACESタグ |
| **Packages** | PACKAGESタグ |
| **Mobile** | CLIENT APPSタグのうちGitHub Mobileアプリに関するもの |
| **Client apps** | CLIENT APPSタグのうちGitHub Desktop・IDE拡張・その他クライアントアプリ |
| **Security** | APPLICATION SECURITY および SUPPLY CHAIN SECURITYタグ |
| **Administration & Enterprise** | ENTERPRISE MANAGEMENT TOOLSタグ |
| **Ecosystem & Accessibility** | ECOSYSTEM & ACCESSIBILITYタグ |
| **Platform governance** | PLATFORM GOVERNANCEタグ |
| **Account management** | ACCOUNT MANAGEMENTタグ |

1つのエントリーが複数タグを持つ場合は、主題として最も関連性の高いカテゴリに分類してください。

## Step 3: リアクションの付与

各エントリーに以下のルールで絵文字リアクションを1つ付与してください（優先順位順）:

1. **🌇**: RETIREDタイプのエントリー（廃止・サービス終了）— 最優先
2. **📍**: RELEASEタイプのエントリー（新機能・新ツールのGA/公開）
3. **👀**: IMPROVEMENTタイプのエントリー（既存機能の改善・アップデート）
4. **🔧**: メンテナンス・設定変更・インフラ関連のエントリー

## Step 4: Discussion本文の作成

以下のフォーマットで Discussion 本文を作成してください。

**[本文ヘッダー]**

まず以下の形式でヘッダーを作成します（`[title]` は入力値で置換、`[period]` は期間）:

```
## [title]

Radio配信のアジェンダです。

• 期間: [start_dateをYYYY.M.D形式]-[end_dateをYYYY.M.D形式]

個人的判断により、以下のリアクションをつけています。

• 📍: 配信でとりあげたい
• 👀: 気になる
• 🌇: サービスの終了、廃止など
• 🔧: メンテナンス
```

**[Changelogセクション]**

次に以下のセクションを続けます:

```
## GitHub Changelog

GitHub Changelogの各記事をカテゴリに分け、リアクション振りました。

### Copilot

（エントリーをここに列挙）

### Models

（エントリーをここに列挙）

### Project & Issues

（エントリーをここに列挙）

...以降すべてのカテゴリ
```

**フォーマット規則**:

- 日付は `YYYY.M.D` 形式（ゼロ埋めなし）で記載する（例: `2026.2.16`）
- 期間は `start_date形式-end_date形式` で記載する（例: `2026.2.16-2026.2.28`）
- エントリーのないカテゴリも `### カテゴリ名` の見出しは省略しない（空のセクション）
- エントリーは以下の書式で記載する:
  `• [エントリータイトル](エントリーURL) 絵文字`
- バレットポイントは `•`（U+2022）を使用する（`-` や `*` は使わない）
- リアクション絵文字はリンクの後にスペース1つを置いて表記する
- すべてのカテゴリを以下の順序で列挙する:
  1. Copilot
  2. Models
  3. Project & Issues
  4. Collaboration tools & Community engagement
  5. Actions
  6. Codespaces
  7. Packages
  8. Mobile
  9. Client apps
  10. Security
  11. Administration & Enterprise
  12. Ecosystem & Accessibility
  13. Platform governance
  14. Account management

## Step 5: Discussionの投稿

`create-discussion` safe output を使用して Discussion を作成してください:

- **タイトル**: ${{ github.event.inputs.title }}
- **本文**: Step 4で作成した内容
- **カテゴリ**: ${{ github.event.inputs.category }}（空の場合は `announcements` を使用）

指定期間にエントリーが1件も見つからなかった場合は `noop` safe output を使用し、その旨を記録してください。
