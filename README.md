# XDC Plugin Node Monitor

Grafana + Prometheus による XDC Plugin Node 監視スタック。

## 構成

```
Plugin Node x4 (:6689/metrics)
        ↓ pull (30秒ごと)
  Prometheus (:9090)   ← メトリクス収集・保存・クエリ
        ↓
    Grafana (:8090)    ← ダッシュボード表示
        ↓
  nginx (80/443)       ← リバースプロキシ (HTTPS)
        ↓
     ブラウザ
```

### Prometheus

メトリクスの収集・保存・クエリを担当。ブラウザから `http://localhost:9090` でExplorer（管理画面）にアクセスでき、PromQLでメトリクスを直接クエリできる。外部には公開せず、SSHトンネル経由でアクセスする。

### Grafana

Prometheusのデータを可視化するダッシュボード。nginxのリバースプロキシ経由で `https://your.domain.example` からアクセスする。

## SSHトンネルでのアクセス

PrometheusのExplorerは外部に公開しないため、SSHトンネルでローカルPCから安全にアクセスする。

```bash
# ローカルPCで実行
ssh -L 9090:localhost:9090 ユーザー名@監視VPSのIP
```

接続後、ローカルのブラウザで `http://localhost:9090` を開く。

> GrafanaはSSHトンネルでもアクセス可能。nginxのセットアップ前の動作確認や、外部に公開せず使いたい場合は以下を使う：
>
> ```bash
> ssh -L 8090:localhost:8090 ユーザー名@監視VPSのIP
> ```
>
> 接続後、ローカルのブラウザで `http://localhost:8090` を開く。

## 監視対象ノード

監視対象のIPアドレスは `prometheus/prometheus.yml` に直接記載してください（テンプレート: `prometheus/prometheus.yml.example`）。

## セットアップ

### 1. Dockerのインストール（未インストールの場合）

```bash
curl -fsSL https://get.docker.com | sh
```

インストール後、現在のユーザーをdockerグループに追加する。これをしないと `docker` コマンド実行時に `permission denied` エラーが発生する。

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> `newgrp docker` が効かない場合は一度ログアウト→ログインし直す。

### 2. Prometheusの設定

```bash
cp prometheus/prometheus.yml.example prometheus/prometheus.yml
```

`prometheus/prometheus.yml` を編集して各ノードのIPアドレスを記載する。

### 3. Grafanaパスワードの変更

`docker-compose.yml` の以下の行を編集：

```yaml
- GF_SECURITY_ADMIN_PASSWORD=your_password_here
```

### 4. Docker Hubにログイン

Docker HubはIPアドレス単位でpullレート制限をかけている。VPSの場合、以前の利用者の使用履歴が残っていることがあり、初回でも制限に引っかかる場合がある。ログインすることで制限を回避できる。

[hub.docker.com](https://hub.docker.com) でアカウント作成後：

```bash
docker login
```

### 5. 起動

```bash
docker compose up -d
```

### 6. nginx + Let's Encrypt の設定

#### Certbotのインストール（未インストールの場合）

```bash
sudo apt install -y certbot python3-certbot-nginx
```

#### nginx設定ファイルを作成（port 80のみ）

```bash
# テンプレートからコピーしてドメインを設定
cp nginx/plugin-monitor.conf.example nginx/plugin-monitor.conf
sed -i 's/your.domain.example/実際のドメイン/g' nginx/plugin-monitor.conf

# plugin-monitorのファイルをsites-availableにシンボリックリンク
sudo ln -s $(pwd)/nginx/plugin-monitor.conf /etc/nginx/sites-available/plugin-monitor

# sites-enabledに有効化
sudo ln -s /etc/nginx/sites-available/plugin-monitor /etc/nginx/sites-enabled/plugin-monitor

# 文法チェックしてリロード
sudo nginx -t && sudo systemctl reload nginx
```

#### Certbotを実行（nginx設定が自動でHTTPS対応に書き換えられる）

```bash
sudo certbot --nginx -d your.domain.example

# 自動更新の確認
sudo certbot renew --dry-run
```

### 7. Grafanaへアクセス

`https://your.domain.example` をブラウザで開く。

- ユーザー名: `admin`
- パスワード: 手順3で設定したもの

ダッシュボードは `Plugin Nodes > XDC Plugin Node Monitor` に自動で読み込まれます。

### 8. SSL証明書の自動更新確認

```bash
sudo certbot renew --dry-run
```

## ファイアウォール設定（各Plugin NodeのVPS）

監視VPSのIPからのみ6689ポートを許可：

```bash
ufw allow from <監視VPSのIP> to any port 6689
```

## ダッシュボード内容

- **Node Status** — ノードの死活監視（UP/DOWN）
- **XDC Balance** — 各ノードのXDC残高（現在値・推移グラフ）
- **RPC Call Failure Rate** — RPCコール失敗率 (%)
- **RPC Dial Failure Rate** — 接続失敗率 (%)
- **DB Connections** — データベース接続数
- **Network Connectivity Failures** — ネットワーク障害累積カウント
