# FigmaデザインからWebアプリデプロイ

## 概要

Figma MCPサーバーでデザインを取得し、React（Vite）フロントエンドコードを生成、Docker ComposeでパッケージしてConoHa VPSにデプロイするワークフロー。デザイナーがFigmaで作成したデザインを、開発者がClaude Codeで迅速にWebアプリとしてデプロイする場合に使用する。

## 基本構成

- **ソース**: Figma MCPサーバー
- **フロントエンド**: React（Vite）+ Nginx（静的ファイル配信）
- **ノード数**: 1
- **OS**: Ubuntu
- **コンテナ構成**: マルチステージビルド（npm build → Nginx）

```
[Figma] -- MCP --> [Claude Code] -- コード生成 --> [ローカルPC]
                                                      ├── src/
                                                      ├── Dockerfile
                                                      └── docker-compose.yml
                                                           |
                                                      tar+SSH
                                                           |
                                                      [ConoHa VPS]
                                                      └── /opt/conoha/<app-name>/
```

## 前提条件

- Figmaアカウントとパーソナルアクセストークンがあること
- Figma MCPサーバーがClaude Codeに設定済みであること（`~/.claude/settings.json`）
- `conoha-cli` がインストール済み・認証済みであること
- SSHキーペアが登録済みであること

> **注意**: Figma MCPサーバーは外部ツールであり、動作の安定性はMCPサーバーの実装に依存する。事前に接続テストを行うことを推奨する。

## 手順

### 1. 事前準備

Figma MCPサーバーの接続を確認する。Claude CodeでFigma MCPのツール一覧が取得できることを確認する。

ConoHaの認証とキーペアを確認する：

```bash
conoha auth login
conoha keypair list
```

対象のFigmaファイルURLとフレーム名を確認する。FigmaファイルURLの形式は `https://www.figma.com/design/<file-key>/<file-name>` である。

### 2. Figmaデザイン取得

Figma MCPを使用してファイル構造を取得する：

- ファイル内のページ・フレーム一覧を取得する
- 対象フレーム（トップレベルのページデザイン）を特定する
- デザイントークンを抽出する：カラーパレット、フォント、スペーシング、レイアウト構造

### 3. コード生成

React（Vite）プロジェクトを生成する：

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
```

Figmaデザインから以下を生成する：

- Reactコンポーネント（Figmaのフレーム構造に対応）
- CSSスタイル（デザイントークンをCSS変数として定義）
- レスポンシブレイアウト対応
- `index.html` のメタタグ設定

### 4. Docker Compose構成

`Dockerfile`（マルチステージビルド）を作成する：

```dockerfile
FROM node:lts-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`nginx.conf` を作成する（SPAルーティング対応）：

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

`docker-compose.yml` を作成する：

```yaml
services:
  web:
    build: .
    ports:
      - "80:80"
    restart: unless-stopped
```

### 5. ConoHaデプロイ

サーバーを作成する（既存サーバーがある場合はスキップ）：

```bash
conoha server create --name my-figma-app --flavor <フレーバーID> --image <UbuntuイメージID> --key-name <キー名> --wait
```

Docker環境を初期化してデプロイする：

```bash
conoha app init my-figma-app --app-name my-app
conoha app deploy my-figma-app --app-name my-app
```

### 6. 動作確認

コンテナの状態を確認する：

```bash
conoha app status my-figma-app --app-name my-app
```

ブラウザでサーバーIPにアクセスし、以下を確認する：

- ページが正常に表示されること
- Figmaデザインとの目視比較で大きな差異がないこと
- レスポンシブ表示が正しく機能すること

## カスタマイズ

### フレームワーク変更

React以外のフレームワークを使用する場合：

| フレームワーク | コマンド |
|---------------|---------|
| Next.js | `npx create-next-app@latest my-app` |
| Vue (Vite) | `npm create vite@latest my-app -- --template vue` |
| Nuxt | `npx nuxi@latest init my-app` |

Next.jsの場合はDockerfileを変更し、`npm run build && npm start` でNode.jsサーバーを起動する構成にする。

### SSL/TLS対応

Let's EncryptでSSL証明書を取得する場合、docker-compose.ymlにcertbotサービスを追加する。

### カスタムドメイン

ConoHa DNSでドメインを設定する：

```bash
conoha dns zone create --name example.com
conoha dns record create --zone-id <ゾーンID> --name www --type A --data <サーバーIP>
```

## トラブルシューティング

| 問題 | 対処 |
|------|------|
| Figma MCPに接続できない | APIトークンの有効性を確認する。`~/.claude/settings.json` のMCPサーバー設定を確認する |
| `npm run build` が失敗する | Node.jsバージョンを確認する（LTS推奨）。依存パッケージの競合を `npm ls` で確認する |
| デプロイ後にページが表示されない | `conoha app logs my-figma-app --app-name my-app` でエラーを確認する |
| Nginx 404エラー | `nginx.conf` の `try_files` 設定を確認する。SPA対応が必要 |
| セキュリティグループでブロック | ポート80（HTTP）が開放されているか確認する |
