# QRV MCP設定ガイド（Claude Desktop利用者向け）

**文書バージョン:** 1.0
**作成日:** 2025-01-27
**対象ユーザー:** QRVサービスを利用する依頼者（Claude Desktopユーザー）
**関連文書:** [ユーザーマニュアル_依頼者向け.md](ユーザーマニュアル_依頼者向け.md)

---

## 目次

1. [はじめに](#1-はじめに)
2. [事前準備](#2-事前準備)
3. [MCP設定手順](#3-mcp設定手順)
4. [QRV MCPサーバーの接続確認](#4-qrv-mcpサーバーの接続確認)
5. [初回予約の実行](#5-初回予約の実行)
6. [トラブルシューティング](#6-トラブルシューティング)
7. [よくある質問（FAQ）](#7-よくある質問faq)

---

## 1. はじめに

### 1.1 本ガイドについて

このガイドでは、Claude DesktopでQRV（Qualfia Reservation）サービスを利用するためのMCP（Model Context Protocol）設定方法を説明します。

### 1.2 MCPとは

MCP（Model Context Protocol）は、AIエージェント（Claude、ChatGPT等）が外部サービスと連携するためのプロトコルです。MCPを設定することで、Claude Desktopから直接QRVサービスを利用できます。

### 1.3 設定後にできること

- Claude Desktopでチャットしながら予約依頼
- 予約状況の確認
- 予約IDの取得とSMS通知の受信

---

## 2. 事前準備

### 2.1 必要なもの

以下のものをご用意ください：

- **Claude Desktop**: 最新バージョン（MCPサポート版）
- **QRV MCPサーバーのURL**: Qualfiaから提供される接続情報
- **認証トークン**: Qualfiaから提供されるAPIトークン
- **携帯電話**: SMS受信可能な携帯電話番号

### 2.2 Claude Desktopのインストール

Claude Desktopがインストールされていない場合は、以下からダウンロードしてください：

**公式サイト:**
```
https://claude.ai/download
```

**対応OS:**
- Windows 10/11
- macOS 10.15以降
- Linux（Ubuntu 20.04以降）

### 2.3 QRV接続情報の取得

Qualfiaから以下の接続情報を取得してください：

1. **MCPサーバーURL**: 例）`https://qrv.qualfia.com/mcp`
2. **認証トークン**: 例）`qrv_token_xxxxxxxxxxxxx`

**注:** 接続情報は初回登録時にメールで送付されます。紛失した場合は inquiry@qualfia.com までお問い合わせください。

---

## 3. MCP設定手順

### 3.1 Claude Desktop設定ファイルの場所

Claude Desktopの設定ファイルは、OSごとに以下の場所にあります：

**Windows:**
```
%APPDATA%\Claude\claude_desktop_config.json
```

**macOS:**
```
~/Library/Application Support/Claude/claude_desktop_config.json
```

**Linux:**
```
~/.config/Claude/claude_desktop_config.json
```

### 3.2 設定ファイルの編集

#### ステップ1: Claude Desktopを終了

設定ファイルを編集する前に、Claude Desktopを完全に終了してください。

#### ステップ2: 設定ファイルを開く

テキストエディタ（メモ帳、VS Code等）で設定ファイルを開きます。

**Windows（メモ帳で開く）:**
1. Windowsキー + R を押す
2. `%APPDATA%\Claude\claude_desktop_config.json` と入力してEnter
3. メモ帳で開く

**macOS（Finderで開く）:**
1. Finder → 移動 → フォルダへ移動
2. `~/Library/Application Support/Claude/` と入力
3. `claude_desktop_config.json` をテキストエディタで開く

#### ステップ3: MCP設定を追加

設定ファイルに以下のMCP設定を追加します。

**設定例:**
```json
{
  "mcpServers": {
    "qrv": {
      "command": "npx",
      "args": [
        "-y",
        "@qualfia/qrv-mcp-server"
      ],
      "env": {
        "QRV_API_URL": "https://qrv.qualfia.com/api",
        "QRV_API_TOKEN": "qrv_token_xxxxxxxxxxxxx"
      }
    }
  }
}
```

**重要な設定項目:**

| 項目 | 説明 | 例 |
|------|------|-----|
| `command` | 実行コマンド | `npx`（固定） |
| `args` | パッケージ名 | `@qualfia/qrv-mcp-server`（固定） |
| `QRV_API_URL` | QRV APIのURL | Qualfiaから提供 |
| `QRV_API_TOKEN` | 認証トークン | Qualfiaから提供 |

**注意:**
- `QRV_API_TOKEN` の値は、Qualfiaから提供されたトークンに置き換えてください
- JSON形式を崩さないように注意してください（カンマ、括弧等）

#### ステップ4: 設定ファイルを保存

編集した設定ファイルを保存します。

**保存時の注意:**
- 文字コードは UTF-8 で保存
- 改行コードは LF または CRLF
- ファイル名は変更しない

#### ステップ5: Claude Desktopを再起動

Claude Desktopを起動すると、MCP設定が読み込まれます。

---

## 4. QRV MCPサーバーの接続確認

### 4.1 MCP接続状態の確認

Claude Desktopを起動後、以下の方法でMCP接続状態を確認できます。

**確認方法:**
1. Claude Desktopを起動
2. 設定画面またはステータスバーでMCP接続状態を確認
3. 「qrv」が接続済みと表示されていればOK

**接続エラーの場合:**
- 設定ファイルのJSON形式が正しいか確認
- 認証トークンが正しいか確認
- インターネット接続を確認

### 4.2 テストメッセージ

Claude Desktopのチャットで以下のメッセージを送信して、QRV MCPサーバーが正常に動作しているか確認します。

**テストメッセージ例:**
```
QRVサービスに接続できていますか？
```

**正常な場合の応答例:**
```
はい、QRV（Qualfia Reservation）サービスに接続されています。
予約の作成や予約状況の確認ができます。
```

**接続エラーの場合の応答例:**
```
QRVサービスに接続できません。MCP設定を確認してください。
```

---

## 5. 初回予約の実行

### 5.1 予約依頼の例

MCP設定が完了したら、実際に予約を依頼してみましょう。

**予約依頼例（アロママッサージ）:**
```
明日の午後、渋谷でアロママッサージを予約したい。
住所: 渋谷区道玄坂1-2-3
時間帯: 14時から17時の間
希望時間: 60分
電話番号: 080-1234-5678
```

**予約依頼例（胡蝶蘭配送）:**
```
胡蝶蘭を配送してほしい。
配送先: 港区六本木1-1-1
配送日: 明後日
サイズ: 3本立て
立て札: 「祝 開店」
電話番号: 080-1111-2222
```

### 5.2 予約フロー

1. Claudeが予約内容を確認
2. 内容を承認すると予約IDが発行される
3. SMSで確認リンクが送信される
4. SMS内の「進める」ボタンを押す
5. システムが提携店に自動電話
6. 予約成功後、決済リンクが送信される
7. 決済を完了すると予約確定

詳細な予約フローは[ユーザーマニュアル_依頼者向け.md](ユーザーマニュアル_依頼者向け.md)を参照してください。

---

## 6. トラブルシューティング

### 6.1 「QRVサービスに接続できません」と表示される

**原因と対処法:**

#### 原因1: 設定ファイルのJSON形式が間違っている

**対処法:**
- JSON形式チェッカーで設定ファイルを検証
- カンマ、括弧、引用符が正しいか確認
- オンラインJSON検証ツール: https://jsonlint.com/

#### 原因2: 認証トークンが間違っている

**対処法:**
- Qualfiaから提供された認証トークンを再確認
- トークンをコピー&ペーストして再設定
- トークンの前後にスペースが入っていないか確認

#### 原因3: Node.jsがインストールされていない

**対処法:**
- Node.js 18以降をインストール
- 公式サイト: https://nodejs.org/
- インストール後、Claude Desktopを再起動

#### 原因4: インターネット接続が不安定

**対処法:**
- インターネット接続を確認
- VPNやプロキシ設定を確認
- ファイアウォールがブロックしていないか確認

### 6.2 設定ファイルが見つからない

**対処法:**

**Windows:**
1. エクスプローラーで「表示」→「隠しファイル」を表示に設定
2. `%APPDATA%` フォルダを開く
3. `Claude` フォルダを探す

**macOS:**
1. Finderで Command + Shift + G を押す
2. `~/Library/Application Support/Claude/` と入力

**Linux:**
1. ターミナルで `ls -la ~/.config/Claude/` を実行

### 6.3 「npx コマンドが見つかりません」と表示される

**対処法:**
1. Node.js 18以降をインストール（npxは同梱されています）
2. インストール後、ターミナルで `npx --version` を実行して確認
3. Claude Desktopを再起動

### 6.4 設定ファイルを編集したが反映されない

**対処法:**
1. Claude Desktopを完全に終了（タスクトレイからも終了）
2. 設定ファイルを再確認
3. Claude Desktopを再起動

---

## 7. よくある質問（FAQ）

### Q1. 複数のMCPサーバーを設定できますか？

**A:** はい、可能です。`mcpServers` 内に複数のサーバー設定を追加できます。

**設定例:**
```json
{
  "mcpServers": {
    "qrv": {
      "command": "npx",
      "args": ["-y", "@qualfia/qrv-mcp-server"],
      "env": {
        "QRV_API_URL": "https://qrv.qualfia.com/api",
        "QRV_API_TOKEN": "qrv_token_xxxxxxxxxxxxx"
      }
    },
    "other-service": {
      "command": "npx",
      "args": ["-y", "@other/mcp-server"],
      "env": {
        "API_KEY": "other_token_xxxxxxxxxxxxx"
      }
    }
  }
}
```

### Q2. 認証トークンを変更する方法は？

**A:** 設定ファイルの `QRV_API_TOKEN` の値を新しいトークンに変更し、Claude Desktopを再起動してください。

### Q3. MCPサーバーを一時的に無効化する方法は？

**A:** 設定ファイルから該当のMCPサーバー設定を削除またはコメントアウトし、Claude Desktopを再起動してください。

### Q4. MCP設定はアカウント単位ですか、デバイス単位ですか？

**A:** デバイス単位です。複数のデバイスでClaude Desktopを使用する場合、各デバイスで設定が必要です。

### Q5. Node.jsのバージョンは何が必要ですか？

**A:** Node.js 18以降が必要です。最新のLTS版（Long Term Support）を推奨します。

### Q6. 設定ファイルのバックアップは必要ですか？

**A:** 推奨します。設定を変更する前に、設定ファイルをコピーしてバックアップを取っておくと安全です。

### Q7. Windows Defenderがブロックします

**A:** 初回実行時にWindows Defenderが警告を表示する場合があります。Qualfia公式のMCPサーバーであることを確認の上、「許可する」を選択してください。

---

## 付録

### A. 設定ファイルの完全な例

**Windows向け設定例:**
```json
{
  "mcpServers": {
    "qrv": {
      "command": "npx",
      "args": [
        "-y",
        "@qualfia/qrv-mcp-server"
      ],
      "env": {
        "QRV_API_URL": "https://qrv.qualfia.com/api",
        "QRV_API_TOKEN": "qrv_token_xxxxxxxxxxxxx"
      }
    }
  }
}
```

**macOS/Linux向け設定例:**
```json
{
  "mcpServers": {
    "qrv": {
      "command": "npx",
      "args": [
        "-y",
        "@qualfia/qrv-mcp-server"
      ],
      "env": {
        "QRV_API_URL": "https://qrv.qualfia.com/api",
        "QRV_API_TOKEN": "qrv_token_xxxxxxxxxxxxx"
      }
    }
  }
}
```

### B. 関連リンク

- **Claude Desktop公式サイト**: https://claude.ai/download
- **MCP公式ドキュメント**: https://modelcontextprotocol.io
- **Node.js公式サイト**: https://nodejs.org/
- **QRV公式サイト**: （運用開始時に追加予定）

### C. お問い合わせ

MCP設定に関するご質問は、以下までお問い合わせください：

**メールアドレス:**
```
inquiry@qualfia.com
```

**件名:**
```
QRV MCP設定に関する問い合わせ
```

---

**注記:** 本ガイドは要件定義フェーズでの初版です。実装フェーズで以下の内容を追加予定です：
- スクリーンショット付きの詳細手順
- 動画チュートリアル
- より詳細なトラブルシューティング
- 環境別の設定例

---

**文書終了**
