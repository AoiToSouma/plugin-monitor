# XDC Plugin Node Monitor

Grafana + Prometheus による XDC Plugin Node 監視スタック。

## 構成

- **Prometheus** (`:9090`) — 4台のPlugin Nodeから30秒ごとにメトリクスを収集
- **Grafana** (`:3000`) — ダッシュボード表示

## 監視対象ノード

監視対象のIPアドレスは `prometheus/prometheus.yml` に直接記載してください。

## セットアップ

### 1. Dockerのインストール（未インストールの場合）

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

### 2. パスワードの変更

`docker-compose.yml` の以下の行を編集：

```yaml
- GF_SECURITY_ADMIN_PASSWORD=your_password_here
```

### 3. 起動

```bash
docker compose up -d
```

### 4. Grafanaへアクセス

`http://<監視VPSのIP>:3000` をブラウザで開く。

- ユーザー名: `admin`
- パスワード: 手順2で設定したもの

ダッシュボードは `Plugin Nodes > XDC Plugin Node Monitor` に自動で読み込まれます。

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
