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
| `.env.example`                          | 環境変数のサンプル（必要な変数一覧）   |


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
- `.env`ファイルを作成（`.env.example` をコピーして値を入力）
  ```bash
  cp .env.example .env
  # Cursorで .env を開いて実際の値を入力
  ```
  > `.env` は絶対に `git add` しないこと。`.gitignore` に含まれているか必ず確認する。

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

### Phase 9：SSH鍵認証の設定（推奨）

rootパスワード認証よりセキュアなSSH鍵認証に切り替える。GitHub Actionsでも鍵認証を使うことで、Secretsにパスワードを入れずに済む。

**ローカルで鍵を生成（未作成の場合）：**
```bash
ssh-keygen -t ed25519 -C "vps-deploy"
# 保存先: ~/.ssh/id_ed25519_vps（デフォルトと分けると管理しやすい）
```

**VPSに公開鍵を登録：**
```bash
ssh-copy-id -i ~/.ssh/id_ed25519_vps.pub root@133.18.181.82
# または手動で
cat ~/.ssh/id_ed25519_vps.pub | ssh root@133.18.181.82 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

**接続確認：**
```bash
ssh -i ~/.ssh/id_ed25519_vps root@133.18.181.82
```

**GitHub Secretsに秘密鍵を登録：**
```
VPS_SSH_KEY：~/.ssh/id_ed25519_vps の内容（-----BEGIN～-----END まで全部）
VPS_HOST：133.18.181.82
VPS_USER：root
```
> `VPS_PASSWORD` の代わりに `VPS_SSH_KEY` を使うように `deploy.yml` を更新すること。

---

### Phase 10：GitHub Actions自動デプロイ

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

### Phase 11：最終確認

- `https://{{SUBDOMAIN}}.mago-t.com`でアクセス確認
- ログイン・主要機能の動作確認
- 画像アップロード等の確認
- GitHubにコミット・push完了

---

## ポート番号管理

VPS上で複数のアプリを動かす場合、ポートが重複しないように管理します。


| サービス       | サブドメイン          | ポート  |
| ---------- | --------------- | ---- |
| FaceRecall    | face.mago-t.com | 5000 |
| MagokoroTech  | mago-t.com      | 5001 |
| （次のサービス） | xxx.mago-t.com  | 5002 |


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

### デプロイ失敗時のロールバック

**直前のコンテナに戻す：**
```bash
ssh root@133.18.181.82
cd /var/www/{{APP_NAME}}/{{APP_NAME}}

# 現在のコンテナを停止
docker compose down

# 直前のイメージに戻す（イメージ名を確認）
docker images | head -10

# タグを指定して起動
docker compose up -d --no-build
```

**Gitで前のコミットに戻す：**
```bash
cd /var/www/{{APP_NAME}}/{{APP_NAME}}
git log --oneline -5          # 戻りたいコミットのハッシュを確認
git checkout <commit-hash>    # 該当コミットに切り替え
docker compose up --build -d  # 再ビルド・起動
```

**GitHub Actionsで前のワークフローを再実行：**
- GitHub の Actions タブ → 成功していたワークフローを選択 → "Re-run jobs"

---

## 参考

- KAGOYA VPSコントロールパネル：[https://vps.kagoya.com/](https://vps.kagoya.com/)
- さくらのドメイン管理：[https://secure.sakura.ad.jp/](https://secure.sakura.ad.jp/)
- Supabaseダッシュボード：[https://supabase.com/](https://supabase.com/)
- Auth0ダッシュボード：[https://manage.auth0.com/](https://manage.auth0.com/)
- DNS確認：[https://dnschecker.org/](https://dnschecker.org/)

