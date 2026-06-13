# XDC Plugin Node Monitor

Grafana + Prometheus による XDC Plugin Node 監視スタック。

## 構成

- **Prometheus** (`:9090`) — 4台のPlugin Nodeから30秒ごとにメトリクスを収集
- **Grafana** (`:8090`) — ダッシュボード表示
- **nginx** — リバースプロキシ（HTTPS）

## 監視対象ノード

監視対象のIPアドレスは `prometheus/prometheus.yml` に直接記載してください（テンプレート: `prometheus/prometheus.yml.example`）。

## セットアップ

### 1. Dockerのインストール（未インストールの場合）

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

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

### 4. 起動

```bash
docker compose up -d
```

### 5. nginx + Let's Encrypt の設定

#### Certbotのインストール（未インストールの場合）

```bash
sudo apt install certbot python3-certbot-nginx
```

#### SSL証明書の取得

```bash
sudo certbot --nginx -d your.domain.example
```

#### nginxの設定ファイルを作成・有効化

```bash
# テンプレートからコピーしてドメインを設定
cp nginx/plugin-monitor.conf.example nginx/plugin-monitor.conf
sed -i 's/your.domain.example/実際のドメイン/g' nginx/plugin-monitor.conf

# plugin-monitorのファイルをsites-availableにシンボリックリンク
sudo ln -s $(pwd)/nginx/plugin-monitor.conf /etc/nginx/sites-available/plugin-monitor

# sites-enabledに有効化
sudo ln -s /etc/nginx/sites-available/plugin-monitor /etc/nginx/sites-enabled/plugin-monitor

# 設定確認・反映
sudo nginx -t
sudo systemctl reload nginx
```

### 6. Grafanaへアクセス

`https://your.domain.example` をブラウザで開く。

- ユーザー名: `admin`
- パスワード: 手順3で設定したもの

ダッシュボードは `Plugin Nodes > XDC Plugin Node Monitor` に自動で読み込まれます。

### 7. SSL証明書の自動更新確認

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
