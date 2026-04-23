# VPS移行テンプレート

## mago-t.com / KAGOYA CLOUD VPS (133.18.181.82)

ReplitからKAGOYA VPSへの移行手順テンプレートです。
FaceRecall移行作業（2026/04/22）をベースに作成。

---

## 前提環境


| 項目             | 内容                   |
| -------------- | -------------------- |
| VPS            | KAGOYA CLOUD VPS 2GB |
| IP             | 133.18.181.82        |
| OS             | Ubuntu 22.04         |
| ドメイン           | mago-t.com（さくらのドメイン） |
| DNS            | さくらのDNS              |
| Docker         | 29.4.1               |
| Docker Compose | v5.1.3               |
| Nginx          | 1.18.0               |


---

## テンプレートファイル一覧


| ファイル                                    | 用途                   |
| --------------------------------------- | -------------------- |
| `Dockerfile.template`                   | Dockerイメージのビルド設定     |
| `docker-compose.yml.template`           | コンテナ起動設定             |
| `.github/workflows/deploy.yml.template` | GitHub Actions自動デプロイ |
| `nginx.conf.template`                   | Nginxリバースプロキシ設定      |


### 変数一覧


| 変数              | 説明            | 例            |
| --------------- | ------------- | ------------ |
| `{{APP_NAME}}`  | アプリ名（ディレクトリ名） | `FaceRecall` |
| `{{PORT}}`      | アプリのポート番号     | `5000`       |
| `{{SUBDOMAIN}}` | サブドメイン名       | `face`       |


---

## 移行チェックリスト

### Phase 1：事前準備

- GitHubリポジトリにコードがpush済みか確認
- `.gitignore`に`.env`が含まれているか確認
- Replitの環境変数（Secrets）一覧をメモ
- ポート番号を確認（デフォルト5000）
- Replitの依存パッケージを確認・削除
  - `@replit/vite-plugin-cartographer`
  - `@replit/vite-plugin-runtime-error-modal`
  - `vite.config.ts`からReplitプラグインを削除
  - `client/index.html`からReplitスクリプトを削除
- Replit Object Storageを使っている場合はSupabase Storageへの移行計画を立てる

### Phase 2：DNS設定

- さくらのドメイン管理画面にログイン
- ゾーン情報 → 編集
- Aレコードを追加
  ```
  ホスト名：{{SUBDOMAIN}}
  タイプ：A
  データ：133.18.181.82
  TTL：3600
  ```
- DNS浸透を確認（dnschecker.orgで確認、5〜30分待つ）

### Phase 3：VPSセットアップ

- PowerShellでSSH接続
  ```bash
  ssh root@133.18.181.82
  ```
- アプリディレクトリを作成
  ```bash
  mkdir -p /var/www/{{APP_NAME}}
  cd /var/www/{{APP_NAME}}
  ```
- GitHubからクローン（Personal Access Tokenが必要）
  ```bash
  git clone https://【TOKEN】@github.com/magotech29/{{APP_NAME}}
  cd {{APP_NAME}}
  git checkout -b vps-deploy
  ```
- `.env`ファイルを作成（Cursorで編集推奨）
  ```
  NODE_ENV=production
  PORT={{PORT}}
  DATABASE_URL=
  AUTH0_DOMAIN=
  AUTH0_CLIENT_ID=
  AUTH0_CLIENT_SECRET=
  AUTH0_AUDIENCE=
  OPENAI_API_KEY=
  SUPABASE_URL=
  SUPABASE_KEY=
  （その他必要な環境変数）
  ```

### Phase 4：Dockerファイル作成

- `Dockerfile`を作成（テンプレートから`{{PORT}}`を置換）
- `docker-compose.yml`を作成（テンプレートから`{{PORT}}`を置換）
- `.dockerignore`を作成
  ```
  node_modules
  .git
  .env
  dist
  *.log
  .DS_Store
  ```

### Phase 5：Nginxの設定

- Nginx設定ファイルを作成
  ```bash
  nano /etc/nginx/sites-available/{{SUBDOMAIN}}.mago-t.com
  ```
  （テンプレートから`{{SUBDOMAIN}}`と`{{PORT}}`を置換して貼り付け）
- シンボリックリンクを作成
  ```bash
  ln -s /etc/nginx/sites-available/{{SUBDOMAIN}}.mago-t.com \
        /etc/nginx/sites-enabled/
  nginx -t
  systemctl reload nginx
  ```

### Phase 6：SSL証明書取得

- DNS浸透済みであることを確認
- Certbotを実行
  ```bash
  certbot --nginx -d {{SUBDOMAIN}}.mago-t.com
  ```
- ブラウザで`https://{{SUBDOMAIN}}.mago-t.com`にアクセスして確認

### Phase 7：アプリのビルド・起動

- Cursor Remote SSHを切断（メモリ節約のため）
- PowerShellからSSH接続してtmuxで実行
  ```bash
  tmux new -s build
  cd /var/www/{{APP_NAME}} && docker compose up --build -d
  ```
- ログを確認
  ```bash
  docker compose logs --tail=30
  ```
- ブラウザで動作確認

### Phase 8：Auth0設定変更（Auth0使用の場合）

- manage.auth0.comにログイン
- アプリのSettings → 以下を追加
  ```
  Allowed Callback URLs：https://{{SUBDOMAIN}}.mago-t.com/callback
  Allowed Logout URLs：https://{{SUBDOMAIN}}.mago-t.com
  Allowed Web Origins：https://{{SUBDOMAIN}}.mago-t.com
  ```
- 既存のReplitのURLはカンマ区切りで残す

### Phase 9：GitHub Actions自動デプロイ

- `.github/workflows/deploy.yml`を作成（テンプレートから`{{APP_NAME}}`を置換）
- GitHubリポジトリのSecretsを設定
  ```
  VPS_HOST：133.18.181.82
  VPS_USER：root
  VPS_PASSWORD：（VPSのrootパスワード）
  ```
  ※Personal Access Tokenに`workflow`スコープが必要
- vps-deployブランチにpush
  ```bash
  git add -A
  git commit -m "Add VPS deployment configuration"
  git push origin vps-deploy
  ```
- GitHub ActionsのActionsタブで成功を確認

### Phase 10：最終確認

- `https://{{SUBDOMAIN}}.mago-t.com`でアクセス確認
- ログイン・主要機能の動作確認
- 画像アップロード等の確認
- GitHubにコミット・push完了

---

## ポート番号管理

VPS上で複数のアプリを動かす場合、ポートが重複しないように管理します。


| サービス       | サブドメイン          | ポート  |
| ---------- | --------------- | ---- |
| FaceRecall | face.mago-t.com | 5000 |
| （次のサービス）   | xxx.mago-t.com  | 5001 |
| （次のサービス）   | xxx.mago-t.com  | 5002 |


---

## トラブルシューティング

### 502 Bad Gateway

```bash
docker compose logs --tail=50
# エラー内容を確認してパッケージ不足等を対処
```

### メモリ不足でビルドが失敗する

```bash
# Cursor Remote SSHを切断してからtmuxでビルド
tmux new -s build
docker compose up --build -d
```

### DNS浸透前にCertbotを実行するとエラー

```
# dnschecker.orgで確認してから再実行
certbot --nginx -d {{SUBDOMAIN}}.mago-t.com
```

### GitHub Actionsで「missing server host」エラー

```
# GitHub SecretsのVPS_HOSTが正しく設定されているか確認
# 1つのSecretに複数の値をまとめて入れないこと
```

---

## 参考

- KAGOYA VPSコントロールパネル：[https://vps.kagoya.com/](https://vps.kagoya.com/)
- さくらのドメイン管理：[https://secure.sakura.ad.jp/](https://secure.sakura.ad.jp/)
- Supabaseダッシュボード：[https://supabase.com/](https://supabase.com/)
- Auth0ダッシュボード：[https://manage.auth0.com/](https://manage.auth0.com/)
- DNS確認：[https://dnschecker.org/](https://dnschecker.org/)

