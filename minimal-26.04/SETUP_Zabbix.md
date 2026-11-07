# Zabbix セットアップ手順 (Ubuntu 26.04 minimal)

Ubuntu 26.04 minimal に Zabbix サーバーを 1 台構成で導入する手順。

| 項目 | 方針 |
| --- | --- |
| Zabbix | 公式リポジトリ (repo.zabbix.com) の最新 LTS |
| 構成 | MySQL + Nginx + PHP-FPM + Zabbix server + Zabbix agent2 を同居 |
| Web UI | 自己署名証明書で HTTPS、80 番は HTTPS へリダイレクト |
| セキュリティ | SSH 鍵認証のみ・root ログイン禁止、ufw で接続元制限、MySQL は localhost のみ |
| 自動更新 | unattended-upgrades で Ubuntu + Zabbix リポジトリを自動適用、必要時 03:30 に自動再起動 |

> 複雑化を避けるため fail2ban、正規 CA 証明書、Zabbix の暗号化通信 (PSK/証明書) は対象外とする。

---

## 0. 前提と変数定義

前提:

- 新規インストール直後の Ubuntu 26.04 minimal
- sudo 可能な一般ユーザーで作業する
- **SSH 公開鍵でログインできることを確認済み** (手順 2 でパスワード認証を無効化するため)

以降のコマンドで使う変数を、作業するシェルで最初に定義する。環境に合わせて値を変更すること。

```bash
# 管理端末 (SSH / Web UI) を許可するサブネット
ADMIN_NET="192.168.1.0/24"
# Zabbix server(10051) へアクティブチェックを送るエージェントのサブネット
AGENT_NET="192.168.1.0/24"
# このサーバーの名前と IP (証明書の SAN に入る)
SERVER_NAME="zabbix.example.lan"
SERVER_IP="192.168.1.10"
# Zabbix の DB パスワード (記号を含めないことで sed での置換事故を防ぐ)
ZBX_DB_PASS="$(openssl rand -hex 24)"
echo "DB password: ${ZBX_DB_PASS}"   # Web 初期設定で使うので控えておく
```

> シェルを開き直した場合は変数を再定義すること (`ZBX_DB_PASS` は控えた値を設定し直す)。

---

## 1. OS 基本設定

```bash
sudo apt update
sudo apt full-upgrade -y

# minimal に含まれない基本ツール
sudo apt install -y vim less curl ca-certificates gnupg openssl

# タイムゾーンと時刻同期
sudo timedatectl set-timezone Asia/Tokyo
sudo systemctl enable --now systemd-timesyncd
timedatectl            # "System clock synchronized: yes" を確認

# (任意) Zabbix UI を日本語表示する場合
sudo apt install -y language-pack-ja
```

---

## 2. 最低限のセキュリティ設定

### 2.1 SSH

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF

sudo sshd -t && sudo systemctl reload ssh
```

> **締め出し防止**: reload 後も今のセッションは切らずに、別ターミナルから鍵で SSH ログインできることを確認してから次に進む。

### 2.2 ファイアウォール (ufw)

SSH の許可ルールを入れてから有効化する。

```bash
sudo apt install -y ufw

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from "${ADMIN_NET}" to any port 22  proto tcp comment 'SSH'
sudo ufw allow from "${ADMIN_NET}" to any port 80  proto tcp comment 'HTTP->HTTPS redirect'
sudo ufw allow from "${ADMIN_NET}" to any port 443 proto tcp comment 'Zabbix UI'
sudo ufw allow from "${AGENT_NET}" to any port 10051 proto tcp comment 'Zabbix trapper'

sudo ufw enable
sudo ufw status verbose
```

> パッシブチェック (server → agent:10050) はサーバーからの送信なので、このサーバー側で追加の許可は不要。監視対象ホスト側で 10050 をこのサーバーから許可すること (付録 A)。

---

## 3. Zabbix リポジトリの追加

1. https://www.zabbix.com/download を開き、以下を選択する。
   - ZABBIX VERSION: 最新の **LTS** (例: 7.0 LTS / 8.0 LTS)
   - OS DISTRIBUTION: Ubuntu / OS VERSION: **26.04**
   - ZABBIX COMPONENT: Server, Frontend, Agent 2 / DATABASE: MySQL / WEB SERVER: Nginx
2. 表示される `zabbix-release_latest_...+ubuntu26.04_all.deb` の URL を控える。
   (メジャー版によりディレクトリ構成が異なるため、URL は必ずページからコピーする)

```bash
ZBX_RELEASE_URL="https://repo.zabbix.com/zabbix/<VER>/.../zabbix-release_latest_<VER>+ubuntu26.04_all.deb"

cd "$(mktemp -d)"
curl -fLO "${ZBX_RELEASE_URL}"
sudo dpkg -i zabbix-release_latest_*.deb
sudo apt update
```

> **Ubuntu 26.04 が選択肢に無い場合**: 公式リポジトリがまだ対応していない。対応を待つか、暫定的に Ubuntu universe の `zabbix-server-mysql` 等を使う (版は古め。その場合この手順のパスは `dpkg -L` で読み替える)。

---

## 4. パッケージの導入

```bash
sudo apt install -y mysql-server
sudo apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf \
                    zabbix-sql-scripts zabbix-agent2
```

PHP-FPM は依存関係で導入される。サービス名を変数に入れておく。

```bash
PHP_FPM="$(systemctl list-unit-files 'php*-fpm.service' --no-legend | awk '{print $1}' | head -n1)"
echo "${PHP_FPM}"      # 例: php8.4-fpm.service
```

---

## 5. MySQL の設定とスキーマ投入

### 5.1 MySQL の待ち受け確認と初期処理

```bash
# 127.0.0.1 のみで待ち受けていることを確認 (Ubuntu の既定値)
sudo grep -rE '^\s*bind-address' /etc/mysql/
sudo ss -tlnp | grep 3306        # 127.0.0.1:3306 のみであること

# 匿名ユーザー・test DB の削除などを対話で実施
#   (root は auth_socket 認証のままでよい / 匿名ユーザー削除: y / リモート root 禁止: y / test DB 削除: y)
sudo mysql_secure_installation
```

### 5.2 DB とユーザーの作成

```bash
sudo mysql <<EOF
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '${ZBX_DB_PASS}';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
SET GLOBAL log_bin_trust_function_creators = 1;
EOF
```

### 5.3 スキーマ投入

スクリプトの配置場所はメジャー版で異なるため、パッケージから場所を取得する。

```bash
ZBX_SQL="$(dpkg -L zabbix-sql-scripts | grep 'mysql/server.sql.gz$')"
echo "${ZBX_SQL}"

zcat "${ZBX_SQL}" | mysql --default-character-set=utf8mb4 -uzabbix -p"${ZBX_DB_PASS}" zabbix
# 数分かかる

sudo mysql -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

---

## 6. Zabbix server / agent2 の設定

```bash
# DB パスワード
sudo sed -i -E "s/^#?\s*DBPassword=.*/DBPassword=${ZBX_DB_PASS}/" /etc/zabbix/zabbix_server.conf
sudo grep -E '^DB(Name|User|Password)=' /etc/zabbix/zabbix_server.conf

# パスワードを含むので他ユーザーから読めないようにする
sudo chown root:zabbix /etc/zabbix/zabbix_server.conf
sudo chmod 640 /etc/zabbix/zabbix_server.conf

# agent2 は自ホスト監視のみ (既定値の確認)
sudo grep -E '^(Server|ServerActive|Hostname)=' /etc/zabbix/zabbix_agent2.conf
#   Server=127.0.0.1 / ServerActive=127.0.0.1 / Hostname=Zabbix server であればよい
```

PHP のタイムゾーンを設定する。

```bash
sudo sed -i -E 's|^;?\s*php_value\[date.timezone\].*|php_value[date.timezone] = Asia/Tokyo|' /etc/zabbix/php-fpm.conf
sudo grep 'date.timezone' /etc/zabbix/php-fpm.conf
```

---

## 7. Nginx と自己署名 HTTPS

### 7.1 自己署名証明書の作成

```bash
sudo openssl req -x509 -newkey rsa:3072 -sha256 -days 825 -nodes \
  -keyout /etc/ssl/private/zabbix.key \
  -out    /etc/ssl/certs/zabbix.crt \
  -subj   "/CN=${SERVER_NAME}" \
  -addext "subjectAltName=DNS:${SERVER_NAME},IP:${SERVER_IP}"
sudo chmod 600 /etc/ssl/private/zabbix.key
```

> 825 日で期限切れになる。期限前に同じコマンドで再作成し `sudo systemctl reload nginx` する。

### 7.2 Zabbix 用 server ブロックを HTTPS 化

`/etc/zabbix/nginx.conf` (= `/etc/nginx/conf.d/zabbix.conf`) のコメントアウトされた `listen` / `server_name` を書き換える。

```bash
sudo sed -i -E \
  -e "s|^#?\s*listen\s+8080;|        listen          443 ssl;|" \
  -e "s|^#?\s*server_name\s+example.com;|        server_name     ${SERVER_NAME};\n        ssl_certificate     /etc/ssl/certs/zabbix.crt;\n        ssl_certificate_key /etc/ssl/private/zabbix.key;|" \
  /etc/zabbix/nginx.conf

sudo sed -n '1,12p' /etc/zabbix/nginx.conf   # listen 443 ssl と ssl_certificate* が入ったことを確認
```

### 7.3 全体設定と HTTP→HTTPS リダイレクト

```bash
# 既定サイト (80 番の Welcome ページ) を無効化
sudo rm -f /etc/nginx/sites-enabled/default

# バージョン非表示と TLS バージョン制限 (http コンテキスト)
sudo tee /etc/nginx/conf.d/00-security.conf >/dev/null <<'EOF'
server_tokens off;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
EOF

# 80 番は HTTPS へリダイレクト
sudo tee /etc/nginx/conf.d/01-redirect.conf >/dev/null <<'EOF'
server {
    listen 80 default_server;
    server_name _;
    return 301 https://$host$request_uri;
}
EOF

sudo nginx -t
```

---

## 8. サービス起動と Web 初期設定

```bash
sudo systemctl restart zabbix-server zabbix-agent2 nginx "${PHP_FPM}"
sudo systemctl enable  zabbix-server zabbix-agent2 nginx "${PHP_FPM}" mysql
```

ブラウザで `https://<SERVER_NAME または SERVER_IP>/` を開く (自己署名のため警告が出るので例外として許可)。

1. **Welcome**: 言語を選択 (日本語は `language-pack-ja` 導入時のみ選択可)
2. **Check of pre-requisites**: すべて OK
3. **Configure DB connection**: Type `MySQL` / Host `localhost` / Database `zabbix` / User `zabbix` / Password `ZBX_DB_PASS` の値
4. **Settings**: Zabbix server name を任意設定、Default time zone `Asia/Tokyo`
5. 完了後、`Admin` / `zabbix` でログイン

ログイン後すぐに実施:

- **Admin のパスワード変更**: User settings → Profile → Change password
- **guest の無効化確認**: Users → Users → `guest` が `Disabled` グループ所属 (既定) であること

---

## 9. 自動更新 (unattended-upgrades)

### 9.1 導入

```bash
sudo apt install -y unattended-upgrades needrestart
```

### 9.2 Zabbix リポジトリの origin を確認

```bash
apt-cache policy | grep -A1 repo.zabbix.com
#   release o=Zabbix,a=...,n=...,l=Zabbix,... のように表示される
#   o= の値 (通常 "Zabbix") を次の設定で使う
```

### 9.3 更新対象・再起動の設定

既定の `50unattended-upgrades` は編集せず、追加ファイルで上書きする (パッケージ更新時の conffile 衝突を避けるため)。

```bash
sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local >/dev/null <<'EOF'
// 更新対象: Ubuntu 本体 (security/updates) + Zabbix 公式リポジトリ
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}";
        "${distro_id}:${distro_codename}-security";
        "${distro_id}:${distro_codename}-updates";
};
Unattended-Upgrade::Origins-Pattern {
        "origin=Zabbix";
};

// 設定ファイルの変更があっても対話待ちで止めず、ローカルの変更を保持する
Dpkg::Options {
        "--force-confdef";
        "--force-confold";
};

Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// 再起動が必要な更新 (カーネル等) の後は 03:30 に自動再起動
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:30";
EOF

sudo tee /etc/apt/apt.conf.d/20auto-upgrades >/dev/null <<'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
EOF
```

### 9.4 更新後のサービス自動再起動 (needrestart)

ライブラリ更新後に PHP-FPM / zabbix-server などを自動で再起動させる。

```bash
sudo mkdir -p /etc/needrestart/conf.d
echo "\$nrconf{restart} = 'a';" | sudo tee /etc/needrestart/conf.d/50-autorestart.conf
```

### 9.5 確認

```bash
sudo unattended-upgrade --dry-run --debug 2>&1 | grep -iE 'allowed origins|zabbix'
systemctl list-timers 'apt-daily*'
```

### 9.6 運用上の注意

- 自動で入るのは **同一メジャー版内のマイナー更新** のみ (zabbix-release パッケージがメジャー版のリポジトリに固定されているため)。マイナー更新で DB スキーマ変更が必要な場合は zabbix-server 起動時に自動で適用される。
- 次の LTS への移行は手動で実施する (新しい `zabbix-release` の導入、リリースノート確認、DB バックアップ取得のうえ実施)。
- 更新履歴: `/var/log/unattended-upgrades/unattended-upgrades.log`、`/var/log/apt/history.log`
- `--force-confold` によりローカル変更した設定ファイルは保持される。新しい既定値は `*.dpkg-dist` として置かれるので、メジャー更新時などに差分を確認する。

---

## 10. 動作確認

```bash
# サービス
systemctl is-active zabbix-server zabbix-agent2 nginx mysql "${PHP_FPM}"   # すべて active

# 待ち受けポート: 3306 は 127.0.0.1 のみ / 80, 443, 10050, 10051 が LISTEN
sudo ss -tlnp | grep -E ':(80|443|3306|10050|10051)\b'

# Web
curl -kI  https://localhost/    # 200 OK
curl -sI  http://localhost/     # 301 → https://
curl -kI  https://localhost/ | grep -i '^server:'   # バージョン番号が出ないこと

# ファイアウォール / SSH
sudo ufw status verbose
sudo sshd -T | grep -E '^(passwordauthentication|permitrootlogin|kbdinteractiveauthentication)'   # すべて no

# 自動更新
sudo unattended-upgrade --dry-run --debug 2>&1 | grep -i 'allowed origins'

# Zabbix ログにエラーが無いこと
sudo tail -n 50 /var/log/zabbix/zabbix_server.log
```

Web UI で **Monitoring → Hosts** の `Zabbix server` の可用性 (ZBX) が緑になっていることを確認する。

---

## 付録 A. 監視対象ホストへの agent2 導入 (最小)

監視対象側でも手順 3 と同じ Zabbix リポジトリを追加したうえで:

```bash
sudo apt install -y zabbix-agent2
sudo sed -i -E \
  -e "s/^Server=.*/Server=<ZabbixサーバーのIP>/" \
  -e "s/^ServerActive=.*/ServerActive=<ZabbixサーバーのIP>/" \
  -e "s/^Hostname=.*/Hostname=$(hostname)/" \
  /etc/zabbix/zabbix_agent2.conf
sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2

# ufw を使っている場合
sudo ufw allow from <ZabbixサーバーのIP> to any port 10050 proto tcp comment 'Zabbix agent'
```

Web UI の Data collection → Hosts で、同じ Hostname・IP のホストを作成し、テンプレート `Linux by Zabbix agent` を割り当てる。

## 付録 B. トラブルシュート

| 症状 | 確認先 |
| --- | --- |
| Zabbix server が起動しない | `/var/log/zabbix/zabbix_server.log`、`journalctl -u zabbix-server` |
| Web UI が 502 / 白画面 | `journalctl -u ${PHP_FPM}`、`/var/log/nginx/error.log` |
| DB 接続エラー | `zabbix_server.conf` の `DBPassword`、`/etc/zabbix/web/zabbix.conf.php` |
| エージェントが赤 | `/var/log/zabbix/zabbix_agent2.log`、監視対象側の ufw、`Server=` の値 |
| 自動更新されない | `/var/log/unattended-upgrades/`、`systemctl list-timers 'apt-daily*'` |
