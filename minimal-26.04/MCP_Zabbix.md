# Claude Code から MCP 経由で Zabbix を参照する設定

[SETUP_Zabbix.md](SETUP_Zabbix.md) で構築した Zabbix 8.0 LTS に、公式の MCP サーバー (`zabbix-mcp-server`) を追加し、Claude Code から監視データ (負荷状況・障害) を **参照のみ** できるようにする手順。

- 対象クライアント: LAN 内の管理端末で動かす **Claude Code**
  (claude.ai のカスタムコネクタは Anthropic のクラウドから接続するため、LAN 内の Zabbix には届かない)
- 参考: Zabbix 8.0 マニュアル「MCP server」「Configuration parameters: Zabbix MCP server」「Tools and resources」

## 構成

```
Claude Code (管理端末 CLIENT_IP)
   │ HTTPS :8443   Authorization: Bearer <読み取り専用 API トークン>
   ▼
zabbix-mcp-server (Zabbix サーバー上)      ← 制限 1: AllowedIP、書き込みツールを公開しない
   │ HTTPS   https://<SERVER_NAME>/api_jsonrpc.php
   ▼
Zabbix API (Nginx + PHP)                   ← 制限 2: トークンのユーザーは参照系 API のみ・対象ホストグループのみ
```

- MCP サーバーはトークンを保存せず、そのまま Zabbix API に渡す。**権限はトークンのユーザーで決まる。**
- 誤操作や想定外の指示に備えて、MCP 側 (ツールの公開範囲) と Zabbix 側 (ユーザーロール) の両方で書き込みを止める。

公式 MCP サーバーのツール (8.0):

| 区分 | ツール | 本手順での扱い |
| --- | --- | --- |
| 参照 | `hostgroup_get` `templategroup_get` `host_get` `template_get` `trigger_get` `problem_get` `maintenance_get` | 公開 |
| 参照 | `item_get` `history_get` (最大 7 日分) | 公開 (同梱設定では拒否されているので解除する) |
| 参照 | `lld_get` `interface_get` `macro_get` | 拒否のまま (負荷確認に不要) |
| 書き込み | `problem_acknowledge` `problem_unacknowledge` `problem_close` `problem_annotate` `problem_change_severity` `problem_suppress` `problem_unsuppress` | 拒否 |
| 書き込み | `maintenance_create` `maintenance_update` `maintenance_delete` | 拒否 |

> **制約**: 取得できる履歴は `history_get` の最大 7 日分まで。トレンド (長期の集計値) を取得するツールは無いので、長期の傾向は Web UI のグラフで確認する。

---

## 0. 前提と変数

- SETUP_Zabbix.md で **Zabbix 8.0 LTS** を構築済み (MCP サーバーは同じ版の Zabbix のみ公式サポート)。
- Claude Code を使う端末から Zabbix サーバーへ HTTPS で到達できる。

Zabbix サーバーで作業するシェルで定義する:

```bash
SERVER_NAME="zabbix.example.lan"   # 証明書の SAN に入れた名前 (SETUP_Zabbix.md 手順 0)
SERVER_IP="192.168.1.10"
CLIENT_IP="192.168.1.50"           # Claude Code を使う端末の IP (複数ならサブネットでも可)
```

---

## 1. Zabbix 側: 読み取り専用ユーザーと API トークン (Web UI)

### 1.1 ユーザーロール

**Users → User roles → Create user role**

| 項目 | 設定 |
| --- | --- |
| Name | `MCP read-only` |
| User type | `User` |
| Access to UI elements | Dashboards 以外のチェックを外す (Web UI は使わない想定) |
| Access to API | Enabled |
| API methods | **Allow list** に `*.get` |
| Access to actions | すべてのチェックを外す (`Default access to new actions` も外す) |

### 1.2 ユーザーグループ

**Users → User groups → Create user group**

- Group name: `MCP readers`
- Host permissions: Claude に見せたいホストグループに **Read** を付ける
  (例: `Zabbix servers`、`Linux servers`、`CI/GitLab Runner`)

> 権限を付けていないホストグループのデータは、MCP からも見えない。

### 1.3 ユーザー

**Users → Users → Create user**

- Username: `mcp-claude`
- Groups: `MCP readers`
- Password: 十分に長いランダムな値 (Web UI ログインには使わない)
- Permissions タブ: Role = `MCP read-only`

### 1.4 API トークン

**Users → API tokens → Create API token**

- Name: `claude-code-<端末名>` (端末ごとに分ける)
- User: `mcp-claude`
- Set expiration date and time: **有効にする** (例: 90 日後)

作成時に表示されるトークンは **一度しか表示されない** ので控える。

### 1.5 トークンの権限確認 (Zabbix サーバーで実施)

```bash
read -rs ZBX_TOKEN   # 控えたトークンを入力 (画面に表示されない)

api() {
  curl -s "https://${SERVER_NAME}/api_jsonrpc.php" \
    -H 'Content-Type: application/json-rpc' \
    -H "Authorization: Bearer ${ZBX_TOKEN}" \
    -d "$1"; echo
}

# 参照: ホスト一覧が返ること
api '{"jsonrpc":"2.0","method":"host.get","params":{"output":["host"]},"id":1}'
# 書き込み: "No permissions to call" 系のエラーになること
api '{"jsonrpc":"2.0","method":"event.acknowledge","params":{"eventids":["0"],"action":4,"message":"test"},"id":2}'
```

> 自己署名証明書のため、この時点では curl が証明書エラーになる場合がある。その場合は手順 3.2 を先に実施する。

---

## 2. MCP サーバーの導入

SETUP_Zabbix.md 手順 3 で追加した Zabbix 公式リポジトリから導入する。

```bash
sudo apt install -y zabbix-mcp-server

# 同梱の設定と、サービスの実行ユーザーを確認しておく
sudo grep -vE '^\s*(#|$)' /etc/zabbix/zabbix_mcp_server.conf
systemctl cat zabbix-mcp-server | grep -E '^(User|Group|ExecStart)='
```

> 自動更新: Zabbix 公式リポジトリのパッケージなので、SETUP_Zabbix.md 手順 9 の unattended-upgrades (`origin=Zabbix`) で自動更新され、needrestart で再起動される。

以降、サービスの実行ユーザーを変数に入れておく (確認結果が `zabbix` 以外なら読み替える)。

```bash
MCP_USER="$(systemctl show -p User --value zabbix-mcp-server)"; MCP_USER="${MCP_USER:-zabbix}"
echo "${MCP_USER}"
```

---

## 3. TLS の準備

### 3.1 MCP サーバー用の証明書と鍵

SETUP_Zabbix.md 手順 7.1 で作った自己署名証明書を流用する。鍵は root 専用 (600) なので、MCP サーバーの実行ユーザーだけが読めるコピーを作る。

```bash
sudo install -d -o root -g "${MCP_USER}" -m 750 /etc/zabbix/mcp
sudo install -o root -g "${MCP_USER}" -m 644 /etc/ssl/certs/zabbix.crt   /etc/zabbix/mcp/zabbix.crt
sudo install -o "${MCP_USER}" -g "${MCP_USER}" -m 600 /etc/ssl/private/zabbix.key /etc/zabbix/mcp/zabbix.key
```

> 証明書を作り直したとき (825 日ごと) は、このコピーもやり直して `sudo systemctl restart zabbix-mcp-server` する。

### 3.2 自己署名証明書を OS に信頼させる

MCP サーバーから Zabbix API (`https://${SERVER_NAME}/...`) へ接続するときに証明書エラーにならないよう、OS の信頼済み証明書に追加する (`IgnoreURLCertErrors` は開発用なので使わない)。

```bash
sudo cp /etc/ssl/certs/zabbix.crt /usr/local/share/ca-certificates/zabbix.crt
sudo update-ca-certificates

# SERVER_NAME が DNS で引けない場合は自ホストを登録
getent hosts "${SERVER_NAME}" || echo "127.0.0.1 ${SERVER_NAME}" | sudo tee -a /etc/hosts

curl -sI "https://${SERVER_NAME}/" | head -n1    # -k なしで 200 が返ること
```

---

## 4. MCP サーバーの設定

`/etc/zabbix/zabbix_mcp_server.conf` を編集する。同じパラメーターが 2 行あると起動に失敗するので、既存の行は置き換え、無ければ追記する関数を使う。

```bash
CONF=/etc/zabbix/zabbix_mcp_server.conf
sudo cp -a "${CONF}" "${CONF}.orig"

set_param() {   # set_param KEY VALUE
  if sudo grep -qE "^\s*$1=" "${CONF}"; then
    sudo sed -i -E "s|^\s*$1=.*|$1=$2|" "${CONF}"
  elif sudo grep -qE "^#\s*$1=" "${CONF}"; then
    sudo sed -i -E "0,/^#\s*$1=.*/s||$1=$2|" "${CONF}"
  else
    echo "$1=$2" | sudo tee -a "${CONF}" >/dev/null
  fi
}

set_param FrontendURL     "https://${SERVER_NAME}/api_jsonrpc.php"
set_param AllowedIP       "${CLIENT_IP}"
set_param ListenPort      8443
set_param TLSAccept       cert
set_param TLSCertFile     /etc/zabbix/mcp/zabbix.crt
set_param TLSKeyFile      /etc/zabbix/mcp/zabbix.key
set_param MaxResponseSize 256K
set_param LogType         system
```

ツールの公開範囲を設定する。

```bash
# 同梱設定の item_get / history_get の拒否を外す (負荷確認に必須)
sudo sed -i -E 's/^\s*(DenyTool=(item_get|history_get))\s*$/# \1   # MCP_Zabbix.md で解除/' "${CONF}"

# 書き込み系ツールをすべて拒否 (RE2 は先読み非対応のため列挙)
sudo tee -a "${CONF}" >/dev/null <<'EOF'

### MCP_Zabbix.md: 参照のみ (書き込み系ツールを公開しない)
DenyToolRegexp=^(problem_(acknowledge|unacknowledge|close|annotate|change_severity|suppress|unsuppress)|maintenance_(create|update|delete))$
EOF

# 結果の確認
sudo grep -vE '^\s*(#|$)' "${CONF}"
```

期待する有効行の例:

```
FrontendURL=https://zabbix.example.lan/api_jsonrpc.php
AllowedIP=192.168.1.50
ListenPort=8443
TLSAccept=cert
TLSCertFile=/etc/zabbix/mcp/zabbix.crt
TLSKeyFile=/etc/zabbix/mcp/zabbix.key
MaxResponseSize=256K
LogType=system
DenyTool=lld_get
DenyTool=interface_get
DenyTool=macro_get
DenyToolRegexp=^(problem_(...)|maintenance_(...))$
```

検証して起動する。

```bash
sudo -u "${MCP_USER}" zabbix_mcp_server -c "${CONF}" -T    # 設定の検証
sudo systemctl enable --now zabbix-mcp-server
sudo systemctl restart zabbix-mcp-server
systemctl is-active zabbix-mcp-server
sudo ss -tlnp | grep 8443
```

---

## 5. ufw

```bash
sudo ufw allow from "${CLIENT_IP}" to any port 8443 proto tcp comment 'Zabbix MCP'
sudo ufw status verbose
```

> MCP サーバーにはレート制限が無い。`AllowedIP` と ufw の許可は Claude Code を使う端末だけに絞る。

---

## 6. 疎通確認 (管理端末から curl)

管理端末に証明書をコピーしておく (手順 7 でも使う)。

```bash
# 管理端末で実施
mkdir -p ~/.config/zabbix-mcp && chmod 700 ~/.config/zabbix-mcp
scp <user>@<SERVER_NAME>:/etc/ssl/certs/zabbix.crt ~/.config/zabbix-mcp/zabbix.crt

read -rs ZABBIX_MCP_TOKEN   # トークンを入力
curl -s --cacert ~/.config/zabbix-mcp/zabbix.crt "https://<SERVER_NAME>:8443/mcp" \
  -H "Authorization: Bearer ${ZABBIX_MCP_TOKEN}" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | grep -oE '"name":"[a-z_]+"' | sort -u
```

確認する点:

- `history_get`、`item_get`、`host_get`、`problem_get` が含まれる
- `problem_acknowledge`、`maintenance_create` など書き込み系が **含まれない**

HTTP ステータスの意味 (`curl -o /dev/null -w '%{http_code}\n' ...` で確認):

| コード | 意味 |
| --- | --- |
| 200 | 接続でき、トークンも受け付けられた |
| 400 | `Accept` ヘッダーに必要なメディアタイプが無い |
| 401 | `Authorization` ヘッダーが無い |
| 403 | 接続元が `AllowedIP` に含まれていない |

---

## 7. Claude Code の設定 (管理端末)

### 7.1 証明書とトークン

トークンはファイルに保存し (600)、シェル起動時に環境変数へ読み込む。Claude Code の設定ファイルには平文で書かない。

```bash
read -rs ZABBIX_MCP_TOKEN
( umask 077; printf '%s' "${ZABBIX_MCP_TOKEN}" > ~/.config/zabbix-mcp/token )

cat >> ~/.bashrc <<'EOF'

# Zabbix MCP (MCP_Zabbix.md)
export NODE_EXTRA_CA_CERTS="$HOME/.config/zabbix-mcp/zabbix.crt"
[ -r "$HOME/.config/zabbix-mcp/token" ] && export ZABBIX_MCP_TOKEN="$(cat "$HOME/.config/zabbix-mcp/token")"
EOF
source ~/.bashrc
```

- `NODE_EXTRA_CA_CERTS`: Claude Code に自己署名証明書を信頼させる。
- 他にも社内 CA などを `NODE_EXTRA_CA_CERTS` で指定している場合は、1 つの PEM ファイルにまとめて指定する。

### 7.2 MCP サーバーの登録

ユーザースコープ (全プロジェクト共通) に登録する。`${ZABBIX_MCP_TOKEN}` は **シングルクォートで囲んで文字列のまま登録** し、Claude Code が接続時に環境変数から展開する。

```bash
claude mcp add-json --scope user zabbix \
  '{"type":"http","url":"https://<SERVER_NAME>:8443/mcp","headers":{"Authorization":"Bearer ${ZABBIX_MCP_TOKEN}"}}'

claude mcp list      # zabbix が ✓ Connected になること
```

Claude Code の中では `/mcp` で接続状態とツール一覧を確認できる。

> 環境変数が未設定のまま起動すると、`${ZABBIX_MCP_TOKEN}` が文字列のまま送られて 401 になり、`claude mcp list` / `/mcp` に警告が出る。ターミナルを開き直すか `source ~/.bashrc` してから Claude Code を起動する。

---

## 8. 使い方 (負荷状況の確認)

Claude は通常、次の順でツールを呼ぶ。

1. `host_get` / `hostgroup_get`: 対象ホストを特定する
2. `item_get`: CPU 使用率、Load average、メモリ使用率、ディスク使用率、`gitlab.runner.jobs` などの itemid を調べる
3. `history_get`: 期間を指定して値を取得する (最大 7 日)
4. `problem_get` / `trigger_get`: 現在の障害を確認する

依頼文の例:

- 「Zabbix で、今出ている障害を重要度順に一覧にして」
- 「`CI/GitLab Runner` グループのホストについて、直近 1 時間の CPU 使用率と load average を要約して」
- 「Runner ホストの `gitlab.runner.jobs` と `gitlab.runner.concurrent` を直近 24 時間で比べて、ジョブが詰まっていた時間帯があるか調べて」
- 「全ホストのファイルシステム使用率を確認して、80% を超えているものを挙げて」

ヒント:

- ホスト名やホストグループ名、アイテムのキーを具体的に指定すると、ツールの呼び出し回数が減って速い。
- 応答が `MaxResponseSize` (256K) を超えると切り詰められる。期間を短くするか、ホストやアイテムを絞って依頼する。
- 監視テンプレートのタグ (`component: gitlab-runner` など、[MONITOR_GitLabRunner.md](MONITOR_GitLabRunner.md)) を指定すると絞り込みやすい。

---

## 9. 運用とセキュリティ

- **トークンの更新**: 期限前に UI で新しいトークンを作成 → 管理端末の `~/.config/zabbix-mcp/token` を差し替え → 古いトークンを削除する。
- **漏えい時**: Users → API tokens で該当トークンを即座に Disable / Delete する。
- **見せる範囲**: ユーザーグループ `MCP readers` のホスト権限で調整する。Claude に見せたくないホストグループには権限を付けない。
- **秘密情報**: パスワードなどはユーザーマクロの **Secret text** タイプにする (API で値が返らない)。
- **ログ**: `journalctl -u zabbix-mcp-server`
- **バージョン**: MCP サーバーと Zabbix サーバーは同じ版で使う。Zabbix をメジャー更新するときは MCP サーバーも同時に更新する。

---

## 10. 動作確認

Zabbix サーバー:

```bash
sudo -u "${MCP_USER}" zabbix_mcp_server -c /etc/zabbix/zabbix_mcp_server.conf -T
systemctl is-active zabbix-mcp-server
sudo ss -tlnp | grep 8443
sudo ufw status | grep 8443                 # CLIENT_IP からのみ許可
```

管理端末:

1. 手順 6 の `tools/list` で、書き込み系ツールが含まれないこと
2. 許可していない端末から接続できないこと (ufw で遮断 / 403)
3. Claude Code の `/mcp` で `zabbix` が connected
4. 「Zabbix server ホストの直近 1 時間の CPU 使用率は？」に、`history_get` の結果を使って回答できる
5. 「今出ている障害を確認済みにして」と依頼しても、該当ツールが無く実行されない

---

## 付録. トラブルシュート

| 症状 | 原因と対処 |
| --- | --- |
| 403 | 接続元が `AllowedIP` に無い。NAT や WSL 経由では、サーバーから見た接続元 IP が端末の IP と異なることがある (`journalctl -u zabbix-mcp-server` で確認) |
| 401 | `Authorization` ヘッダーが無い。`echo ${ZABBIX_MCP_TOKEN:+set}` で環境変数を確認 |
| Claude Code で証明書エラー | `NODE_EXTRA_CA_CERTS` が未設定、または URL のホスト名が証明書の SAN に無い (IP で接続する場合は SAN の IP と一致させる) |
| MCP サーバーから API へ接続できない | 手順 3.2 (OS への証明書追加、`/etc/hosts`) を確認。`curl -sI https://${SERVER_NAME}/` を `-k` なしで試す |
| ツールが一覧に出ない | `DenyTool` / `DenyToolRegexp` の設定を確認 |
| ツールは呼べるが結果が空 | `MCP readers` グループに、対象ホストグループの Read 権限が無い |
| 権限エラー (`No permissions`) | ロールの API メソッド許可 (`*.get`) を確認。MCP サーバーが `.get` 以外の API を内部で使っている場合は、`journalctl -u zabbix-mcp-server` でメソッド名を確認し、参照系に限って許可リストへ追加する |
| 応答が途中で切れる | `MaxResponseSize` の上限 (256K)。期間や対象を絞って依頼する |
