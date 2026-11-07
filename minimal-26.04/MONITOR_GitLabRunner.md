# GitLab Runner サーバーの監視設定 (Zabbix)

[SETUP_Zabbix.md](SETUP_Zabbix.md) で構築した Zabbix サーバーから、GitLab Runner (Docker executor) が稼働する Ubuntu 26.04 minimal サーバーを監視する手順。

- テンプレート: [templates/zbx_template_gitlab_runner.yaml](templates/zbx_template_gitlab_runner.yaml)
- 収集したデータを MCP 経由で Claude から参照する設定は対象外 (別の手順書で扱う)。そのための備えとしてタグ・ホストグループ・保存期間を揃えている (手順 6、付録 B)。

## 監視設計

| 監視対象 | 方法 | トリガー (重要度) |
| --- | --- | --- |
| OS (CPU・メモリ・ディスク・NW) | 公式テンプレート `Linux by Zabbix agent` | 既定のもの (ディスク空き不足など) |
| gitlab-runner サービス | agent2 `systemd.unit.info[gitlab-runner.service]` | 3 回連続で active 以外 (High) |
| docker サービス | agent2 `systemd.unit.info[docker.service]` | 3 回連続で active 以外 (High) |
| GitLab への到達性 | agent2 `net.tcp.service[https,{$GITLAB.HOST},443]` | 5 分間接続不可 (Average) |
| ジョブ数 / concurrent 上限 | Runner メトリクス `gitlab_runner_jobs` / `gitlab_runner_concurrent` | 上限に 15 分張り付き (Warning) |
| Runner の error ログ | `gitlab_runner_errors_total{level="error"}` の増加率 | 10 分間継続 (Warning) |
| GitLab API の 4xx/5xx | `gitlab_runner_api_request_statuses_total` の増加率 | 10 分間継続 (Warning) |
| Runner バージョン | `gitlab_runner_version_info` | なし (棚卸し用) |
| メトリクス取得 | 上記メトリクスが届いているか | 10 分間届かない (Warning) |

メトリクスの取得方法:

```
gitlab-runner ──(127.0.0.1:9252/metrics)──> zabbix-agent2 ──(10050)──> Zabbix server
                 Runner ホスト内のみで公開      web.page.get       パッシブチェック
```

- Runner のメトリクスは **127.0.0.1 のみ** で公開し、agent2 がホスト内で取得する。9252 番を外部に開ける必要がない。
- agent2 は 1 分ごとにメトリクス全体を 1 回取得し (マスターアイテム、保存なし)、そこから個々の値を取り出す (依存アイテム)。

採用しないもの:

- **公式テンプレート `Docker by Zabbix agent 2` は既定では使わない。**
  - zabbix ユーザーを docker グループに入れる必要があり、これは実質的に root 権限を与えることになる。
  - ジョブのコンテナは短命なので、コンテナごとのアイテムが大量に作成・削除されて煩雑になる。
  - dockerd の死活とディスク監視で代替する。どうしても必要な場合は付録 A を参照。

---

## 0. 前提と変数

- Runner サーバーは Ubuntu 26.04 minimal。GitLab Runner (Docker executor) は導入・登録済み。
- Runner サーバーで ufw を使っている (SETUP_Zabbix.md 手順 2.2 と同様の構成)。
- Zabbix サーバーの ufw で、Runner サーバーが `AGENT_NET` に含まれている。

Runner サーバーで作業するシェルで定義する:

```bash
ZBX_SERVER_IP="192.168.1.10"        # Zabbix サーバーの IP
GITLAB_HOST="gitlab.example.com"    # Runner が接続する GitLab のホスト名
```

---

## 1. Runner サーバーに zabbix-agent2 を導入

SETUP_Zabbix.md の **手順 3** と同じ方法で、Zabbix サーバーと同じメジャー版の公式リポジトリを追加してから導入する。

```bash
sudo apt install -y zabbix-agent2

sudo sed -i -E \
  -e "s/^Server=.*/Server=${ZBX_SERVER_IP}/" \
  -e "s/^ServerActive=.*/ServerActive=${ZBX_SERVER_IP}/" \
  -e "s/^Hostname=.*/Hostname=$(hostname)/" \
  /etc/zabbix/zabbix_agent2.conf
sudo grep -E '^(Server|ServerActive|Hostname)=' /etc/zabbix/zabbix_agent2.conf

sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2
```

> 自動更新: Runner サーバーでも SETUP_Zabbix.md 手順 9 と同じ unattended-upgrades の設定をすれば、agent2 も自動更新される (`Origins-Pattern` に `origin=Zabbix` を含める)。

---

## 2. Runner サーバーの ufw

```bash
sudo ufw allow from "${ZBX_SERVER_IP}" to any port 10050 proto tcp comment 'Zabbix agent'
sudo ufw status verbose
```

> Docker は iptables を直接書き換えるので、コンテナの公開ポートには ufw が効かない。ただし agent2 はホスト上で動くプロセスなので、このルールは通常どおり効く。

---

## 3. GitLab Runner のメトリクス公開 (localhost のみ)

`/etc/gitlab-runner/config.toml` の **先頭のグローバル部** (`[[runners]]` より前) に `listen_address` を追加する。

```bash
sudo cp -a /etc/gitlab-runner/config.toml /etc/gitlab-runner/config.toml.bak
sudo vim /etc/gitlab-runner/config.toml
```

```toml
concurrent = 4
check_interval = 0
listen_address = "127.0.0.1:9252"   # ← 追加

[session_server]
  ...

[[runners]]
  ...
```

```bash
sudo systemctl restart gitlab-runner

# 127.0.0.1 のみで待ち受けていること
sudo ss -tlnp | grep 9252
# メトリクスが取得できること
curl -s http://127.0.0.1:9252/metrics | grep -E '^gitlab_runner_(concurrent|jobs|version_info|errors_total)'
```

> `gitlab_runner_jobs` や `gitlab_runner_errors_total` は、ジョブやエラーが一度も無いと出力されないことがある。テンプレートでは出力が無い場合は 0 として扱う。

---

## 4. agent からの取得確認 (Zabbix サーバーで実施)

```bash
sudo apt install -y zabbix-get
RUNNER_IP="192.168.1.20"   # Runner サーバーの IP

zabbix_get -s "${RUNNER_IP}" -k 'agent.ping'                                  # 1
zabbix_get -s "${RUNNER_IP}" -k 'systemd.unit.info[gitlab-runner.service]'   # active
zabbix_get -s "${RUNNER_IP}" -k 'systemd.unit.info[docker.service]'          # active
zabbix_get -s "${RUNNER_IP}" -k 'net.tcp.service[https,gitlab.example.com,443]'  # 1
zabbix_get -s "${RUNNER_IP}" -k 'web.page.get[127.0.0.1,/metrics,9252]' | head
```

失敗する場合は Runner サーバーの `/var/log/zabbix/zabbix_agent2.log`、ufw、`Server=` の値を確認する。

---

## 5. テンプレートのインポート

1. Web UI で **Data collection → Templates → Import**
2. `templates/zbx_template_gitlab_runner.yaml` を選択して Import
3. `GitLab Runner by Zabbix agent 2` がグループ `Templates/Applications` に追加されたことを確認

テンプレートの内容:

| アイテム (キー) | 内容 | 保存期間 |
| --- | --- | --- |
| `systemd.unit.info[gitlab-runner.service]` | gitlab-runner の状態 | 履歴 31d |
| `systemd.unit.info[docker.service]` | docker の状態 | 履歴 31d |
| `net.tcp.service[https,{$GITLAB.HOST},443]` | GitLab への接続可否 (1/0) | 履歴 31d / トレンド 365d |
| `web.page.get[127.0.0.1,/metrics,{$RUNNER.METRICS.PORT}]` | メトリクス全体 (マスター) | 保存しない |
| `gitlab.runner.jobs` | 処理中のジョブ数 | 履歴 31d / トレンド 365d |
| `gitlab.runner.concurrent` | concurrent 上限 | 履歴 31d / トレンド 365d |
| `gitlab.runner.errors.rate` | error ログの発生数 / 分 | 履歴 31d / トレンド 365d |
| `gitlab.runner.api.errors.rate` | GitLab API の 4xx/5xx 応答数 / 分 | 履歴 31d / トレンド 365d |
| `gitlab.runner.version` | Runner バージョン | 履歴 31d |

マクロ (ホスト側で上書き可能):

| マクロ | 既定値 | 意味 |
| --- | --- | --- |
| `{$GITLAB.HOST}` | `gitlab.example.com` | 接続先 GitLab のホスト名 (**必ずホスト側で設定**) |
| `{$RUNNER.METRICS.PORT}` | `9252` | `listen_address` のポート |
| `{$RUNNER.JOBS.FULL.TIME}` | `15m` | 上限に張り付いた状態をジョブ詰まりとみなす時間 |
| `{$RUNNER.ERRORS.MAX}` | `0` | error ログの許容発生数 / 分 |

---

## 6. ホスト登録

1. **Data collection → Host groups → Create host group**: `CI/GitLab Runner` を作成
2. **Data collection → Hosts → Create host**
   - Host name: Runner サーバーの `hostname` (agent2 の `Hostname=` と同じ値)
   - Templates: `Linux by Zabbix agent`、`GitLab Runner by Zabbix agent 2`
   - Host groups: `CI/GitLab Runner`
   - Interfaces: Agent / IP = Runner サーバーの IP / Port 10050
   - Macros タブ: `{$GITLAB.HOST}` = 実際の GitLab ホスト名
3. Add

> ホストグループ `CI/GitLab Runner` と、テンプレート内のタグ (`component: gitlab-runner` / `component: docker`、`scope: availability` / `capacity` / `notice`) は、後で API や MCP から対象を絞り込むために使う。Runner を増やすときも同じグループに入れる。

---

## 7. Docker によるディスク逼迫への備え

Docker executor はイメージ、ビルドキャッシュ、ボリュームが `/var/lib/docker` に溜まり、ディスクを圧迫しやすい。

- **検知**: `Linux by Zabbix agent` のファイルシステム監視 (空き容量の Warning / Critical) で検知できる。`/var/lib/docker` を別パーティションにしている場合も自動で検出される。
- **対処 (参考)**: 定期的に不要なデータを削除する運用を検討する。例:
  ```bash
  # 7 日以上使われていない停止コンテナ・イメージ・ビルドキャッシュを削除
  sudo docker system prune -af --filter "until=168h"
  ```
  systemd timer や cron で定期実行するかどうかは、ジョブのキャッシュ効率と相談して決める (本手順では設定しない)。

---

## 8. 動作確認

Runner サーバー:

```bash
sudo ss -tlnp | grep -E ':(9252|10050)\b'   # 9252 は 127.0.0.1、10050 は全体で LISTEN
sudo ufw status verbose                     # 10050 は Zabbix サーバーのみ許可
```

Zabbix Web UI:

1. **Monitoring → Hosts** で Runner ホストの ZBX が緑
2. **Monitoring → Latest data** で、Runner ホスト・タグ `component: gitlab-runner` に絞って各アイテムに値が入っている
   (`gitlab.runner.errors.rate` などの増加率は、2 回目の取得以降に値が入る)
3. 障害検知テスト (影響の少ない時間帯に実施):
   ```bash
   sudo systemctl stop gitlab-runner
   # 約 3 分後に "GitLab Runner: Service is not running" (High) が発生することを確認
   # ("Metrics are not available" は依存関係により抑止される)
   sudo systemctl start gitlab-runner
   # 回復 (RESOLVED) することを確認
   ```

---

## 付録 A. Docker テンプレートを使う場合 (任意・リスクあり)

コンテナやイメージの詳細を監視したい場合は、公式テンプレート `Docker by Zabbix agent 2` を追加する。agent2 が Docker のソケットを読めるように、zabbix ユーザーを docker グループに入れる必要がある。

```bash
sudo usermod -aG docker zabbix
sudo systemctl restart zabbix-agent2
```

> **注意**: docker グループのユーザーは、コンテナ経由でホストの root 権限を取得できる。agent2 が乗っ取られた場合の被害が大きくなるため、本手順の既定構成では採用していない。
> また短命なジョブコンテナが検出のたびにアイテムを作成・削除するので、テンプレートのマクロ (`{$DOCKER.LLD.FILTER.CONTAINER.MATCHES}` など) で対象を絞ること。

## 付録 B. 後で MCP から参照するための前提 (メモ)

MCP 経由で Claude から参照する設定は別の手順書で扱う。本手順の時点で、次のように揃えてある。

- 対象ホストはホストグループ `CI/GitLab Runner` にまとめてある
- アイテムとトリガーにタグ `component` / `scope` を付けてある (`item.get` / `problem.get` で絞り込める)
- 数値アイテムは履歴 31 日・トレンド 365 日を保存する
- アイテムの説明 (description) に意味と単位を書いてある
