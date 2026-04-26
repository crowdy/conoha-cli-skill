# リリーススモーク (post-merge / pre-tag)

## 概要

`crowdy/conoha-cli` の RC ビルドを **実 VPS** に当てて end-to-end 検証するレシピ。CI の e2e (DinD) では `--privileged` が host 制約をマスクするため、UFW / unprivileged_port / cloud-init / TOFU といった "VPS でしか出ない" バグクラスを潰すのが目的。`docs/release-checklist.md §1 + §2` の自動化版。

PR #162 row 7 の検証として開発され、PR #172 が `--cap-add` 単独では非効率という事実を発見し、PR #174 (host sysctl) / PR #173 (UFW) / PR #178 (status) を生み出した経緯がある。同等の発見を未来のために再現できるよう、各ステップで "今 active な失敗モード" を明示している。

## 適用タイミング

- 重要な PR が main にマージされた直後 (特に proxy / app /lifecycle 系)。
- v0.x.0 の RC タグを切る前。
- 新しい OS / Docker イメージで動作させる前。

## 事前確認 (Pre-flight)

スモーク本体に入る前に **必ず** 以下を順に実施。1 つでも失敗するなら本体スモークを **開始しない**。

```bash
# 1. 認証 + API リーチャビリティ
conoha flavor list --format json | head -3   # tenant 確認 + token 期限が切れていないか
conoha keypair list --no-headers | head -1   # 利用可能な keypair が 1 つ以上ある

# 2. ローカル秘密鍵の存在
ls ~/.ssh/conoha_<keyname> 2>&1               # 公開鍵に対応する秘密鍵がローカルにある

# 3. DNS zone 確認 (任意 — 無ければ sslip.io fallback 使用)
conoha dns domain list --format json
# zone があるなら 'dig @8.8.8.8 NS <zone>' で外部に NS が委任されているか確認。
# NXDOMAIN なら未委任 → sslip.io fallback (推奨)
```

DNS が未委任なら `<host>.<vps-ip>.sslip.io` を使う。`*.<ip>.sslip.io` は IP に直接 resolve されるため Let's Encrypt HTTP-01 が成立する。これにより:
- レジストラへの NS 委任作業が不要
- zone の rate limit / quota を消費しない
- PR #170 (DNS list parser uuid バグ) を回避できる

## 既知のイメージ別ハマりどころ

| イメージ | 注意点 | mitigation |
|---|---|---|
| `vmi-docker-29.2-ubuntu-24.04-amd64` | UFW が `policy DROP` で SSH のみ allow / `net.ipv4.ip_unprivileged_port_start=1024` | conoha-cli #173 + #174 が BootScript で自動対応。古い CLI を使う場合は手動で `ufw allow 80,443/tcp; sysctl -w net.ipv4.ip_unprivileged_port_start=0` |
| RHEL 系 (将来対応時) | `firewalld` が UFW の代わりに動く / SELinux | 未検証 — 該当イメージがサポート対象になったら本表に追記 |

## スモーク本体

タイムスタンプを唯一の "session ID" にして、生成する全リソースに同じ識別子を振る。中断・並行実行時の混乱を防ぐ。

```bash
EPOCH=$(date +%s)
SRV_NAME="rc-smoke-${EPOCH}"
RC_DIR="/tmp/conoha-rc-smoke-${EPOCH}"
mkdir -p "$RC_DIR"
echo "$RC_DIR" > /tmp/rc-smoke-dir
echo "$SRV_NAME" > /tmp/rc-smoke-name
```

### 1. VPS 作成

`g2l-t-c3m2` (3 vCPU / 2G RAM / 0G boot vol → +100GB block) は proxy + 多重 compose stack を流すのに余裕のある最小構成。`g2l-t-c1m512` 等の 512MB は Docker pull で OOM になりがち。

```bash
conoha server create --no-input --yes --wait --wait-timeout 8m \
  --name "$SRV_NAME" \
  --flavor 784f1ae8-0bc8-4d06-a06b-2afaa9580e0a \
  --image 722c231f-3f61-4e79-a5a6-c70d6c9ea908 \
  --key-name <keyname> \
  --security-group default \
  --security-group IPv4v6-SSH \
  --security-group IPv4v6-Web \
  --security-group IPv4v6-ICMP \
  --format json | tee /tmp/rc-smoke-create.json
```

セキュリティグループは **4 つ全部必須**:
- `default` — 基本
- `IPv4v6-SSH` — 22
- `IPv4v6-Web` — 80/443
- `IPv4v6-ICMP` — health probe / ping 用

`--security-group` 未指定 + `--no-input` は `interactive selection requires a TTY` で失敗する (#155)。

### 2. IP / ボリューム ID を保存

```bash
SRV_ID=$(jq -r .id /tmp/rc-smoke-create.json)
echo "$SRV_ID" > /tmp/rc-smoke-srv-id
IP=$(conoha server show "$SRV_ID" --format json \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print([a['addr'] for v in d['addresses'].values() for a in v if a['version']==4][0])")
echo "$IP" > /tmp/rc-smoke-ip
```

ボリューム ID を別途保存する必要は無い — `server delete --delete-boot-volume` が attached なボリュームを自動列挙して削除する (teardown 節参照)。

### 3. cloud-init 完了を待つ

```bash
ssh -o StrictHostKeyChecking=accept-new \
    -i ~/.ssh/conoha_<keyname> root@"$IP" \
    'cloud-init status --wait'
```

cloud-init 完了前に `proxy boot` を打つと `Connection refused` 系のエラーが出る。`--wait` を付けるのが確実 (release-checklist row 1)。

### 4. fixture を staging

最小限のマルチホスト (root + expose:api) を sslip でこしらえる。

```bash
cat > "$RC_DIR/conoha.yml" <<EOF
name: rc-smoke
hosts:
  - rc-smoke-root.${IP}.sslip.io
web:
  service: web
  port: 80
health:
  path: /
expose:
  - label: api
    host: rc-smoke-api.${IP}.sslip.io
    service: api
    port: 80
    health:
      path: /
deploy:
  drain_ms: 5000
EOF

cat > "$RC_DIR/docker-compose.yml" <<'EOF'
services:
  web:
    image: nginx:1.27-alpine
    command: >
      sh -c "echo 'rc-smoke-root OK' > /usr/share/nginx/html/index.html
      && exec nginx -g 'daemon off;'"
  api:
    image: nginx:1.27-alpine
    command: >
      sh -c "echo 'rc-smoke-api OK' > /usr/share/nginx/html/index.html
      && exec nginx -g 'daemon off;'"
EOF
```

### 5. proxy boot — healthy gate を **必ず** 確認

```bash
conoha proxy boot --insecure -i ~/.ssh/conoha_<keyname> \
  --acme-email ops@example.com "$SRV_NAME"
```

期待される出力:
```
==> Done. Admin socket: /var/lib/conoha-proxy/admin.sock
==> Waiting for proxy to become healthy (up to 30s)
Boot complete.
```

`==> Waiting for proxy to become healthy` 行が**出ない**なら、conoha-cli が PR #177 を含まない古いバージョン。**そのまま進めず**にバージョンを上げる。これが PR #172 のような誤った "成功" の最初の防衛線 (#175)。

healthy gate がタイムアウトしたら、エラーに `docker logs --tail 20` の最後 20 行が含まれる。`bind: permission denied` / `dial unix admin.sock: connect: no such file or directory` 等のシグネチャから対応する known-issue を当てはめる。

### 6. init → deploy → status の自動検証

```bash
cd "$RC_DIR"
conoha app init --insecure -i ~/.ssh/conoha_<keyname> "$SRV_NAME"
conoha app deploy --insecure -i ~/.ssh/conoha_<keyname> --slot blue "$SRV_NAME"

# project dir からの status (full report — root + expose)
conoha app status --no-input --insecure -i ~/.ssh/conoha_<keyname> --format json "$SRV_NAME" \
  | jq '{root: .root.phase, expose: [.expose[] | {label, phase: .service.phase}]}'

# /tmp からの status (#178: project file 不要、root のみ + stderr warning)
cd /tmp
conoha app status --no-input --insecure -i ~/.ssh/conoha_<keyname> --app-name rc-smoke --format json "$SRV_NAME" 2>/tmp/status-stderr.log \
  | jq '{root: .root.phase, expose}'
grep -q 'no conoha.yml in cwd' /tmp/status-stderr.log && echo "OK: #178 warning" || echo "MISS: #178 stderr warning"
```

### 7. HTTPS + LE 検証

```bash
ROOT="rc-smoke-root.${IP}.sslip.io"
API="rc-smoke-api.${IP}.sslip.io"

# UFW + sysctl が boot で適用済みなら LE は backoff なし、初回で発行される
for i in $(seq 1 8); do
  code=$(curl -sS -o /dev/null --max-time 10 -w '%{http_code}' "https://${ROOT}/")
  [ "$code" = "200" ] && break
  sleep 8
done

curl -sS --max-time 10 "https://${ROOT}/"   # rc-smoke-root OK
curl -sS --max-time 10 "https://${API}/"    # rc-smoke-api OK

# LE production (E5/E6/E7/E8 系) であることを確認 (staging だと E1)
echo | openssl s_client -connect "${ROOT}:443" -servername "$ROOT" 2>/dev/null \
  | openssl x509 -noout -issuer
```

ACME backoff (~120 秒) が発生したら UFW か sysctl のどちらかが効いていない。boot スクリプトを更新する版が必要。

### 8. rollback の二経路

```bash
cd "$RC_DIR"
# 8a. drain 窓内 — 正常 swap
conoha app deploy --insecure -i ~/.ssh/conoha_<keyname> --slot green "$SRV_NAME"
conoha app rollback --insecure -i ~/.ssh/conoha_<keyname> --drain-ms 2000 "$SRV_NAME"
# expect: "Rollback complete for rc-smoke. ..." + "Rollback complete for rc-smoke-api. ..."

# 8b. drain 窓外 — default は warning + exit 0、--target=web は fatal
sleep 5
conoha app rollback --insecure -i ~/.ssh/conoha_<keyname> "$SRV_NAME" 2>&1 | tee /tmp/rb-default.log
grep -q 'drain window expired' /tmp/rb-default.log && echo "OK: default warning"

conoha app rollback --insecure -i ~/.ssh/conoha_<keyname> --target=web "$SRV_NAME" 2>&1 | tee /tmp/rb-target.log
grep -q 'drain window has closed' /tmp/rb-target.log && echo "OK: --target=web fatal"
```

## バグ探し姿勢 (積極的に観察するもの)

スモーク中、各ステップで以下のシグネチャを **能動的に** チェック。"動いた" だけで通過させない。

| step | チェック対象 |
|---|---|
| 5 | `Boot complete.` の前に `Waiting for proxy to become healthy` が出るか / docker logs に `permission denied` がないか |
| 5 | `ufw status` を ssh で叩いて 80,443 が allow になっているか |
| 5 | `/etc/sysctl.d/99-conoha-proxy.conf` が存在するか |
| 6 | `app status` の JSON で expose 配列に登録した label が **すべて** 出ているか |
| 6 | `/tmp` からの status で **stderr** に warning が出ているか / **stdout** は valid JSON か |
| 7 | LE issuer が `R3` / `E1`–`E8` (production) か、`R10` / staging だと NG |
| 7 | HTTP → HTTPS リダイレクトが 301 で返るか (304 / 200 だとプロキシが TLS 経由なし) |
| 8 | rollback で active が前 slot のポートに戻ったか (URL の port 番号で確認) |

何か逸脱したら **即座に** GitHub Issue 草稿:
```bash
gh issue create --repo crowdy/conoha-cli \
  --title "<エラーの要約>" \
  --label bug \
  --body "Reproduced during release smoke at $(date -Is). Repro: ..."
```

## Teardown — 残骸ゼロ確認まで

`destroy` → `proxy remove --purge` → `server delete --delete-boot-volume --wait` → DNS レコード削除 (sslip 利用なら不要) → ローカルディレクトリ削除。

`server delete --delete-boot-volume` は server 削除 → volume の `detaching` → `available` 遷移待ち → `volume delete` を 1 コマンドで実行する (closed #88 / commit `54db033`)。`--wait` は `--delete-boot-volume` 指定時に自動有効化されるので明示しなくても動くが、明示する方が意図が明確。

```bash
SRV_ID=$(cat /tmp/rc-smoke-srv-id)

cd "$RC_DIR"
conoha app destroy --yes --no-input --insecure -i ~/.ssh/conoha_<keyname> "$SRV_NAME"
conoha proxy remove --purge --insecure -i ~/.ssh/conoha_<keyname> "$SRV_NAME"
conoha server delete --yes --delete-boot-volume --wait "$SRV_ID"

# 残骸が無いことを確認 (これを怠ると課金継続 / dangling リソースが累積する)
# server delete --delete-boot-volume が成功していれば両方とも 0 件のはず。
# 0 件でない場合は --wait のタイムアウトに引っかかったか、別 namespace に
# 残ったボリュームが存在する可能性。手動で確認する。
conoha server list --format json | jq -e --arg id "$SRV_ID" '[.[] | select(.id == $id)] | length == 0' || echo "FAIL: server still listed"
conoha volume list --format json | jq -e --arg name "$SRV_NAME" '[.[] | select(.name | startswith($name))] | length == 0' || echo "FAIL: smoke volume(s) still listed"

cd / && rm -rf "$RC_DIR" /tmp/rc-smoke-*
```

中断時 (Ctrl-C 等) も上記 teardown は **必ず** 実行する。`/tmp/rc-smoke-*` ファイルが ID を保持しているので残骸を確実に追える。

## 結果レポーティング (PR / リリースノートに貼る)

```markdown
## Manual VPS smoke (release-checklist §2 row 7) — ✅ pass / ❌ fail

Ran on a fresh `g2l-t-c3m2` VPS using `vmi-docker-29.2-ubuntu-24.04-amd64`,
multi-host fixture (root + expose:api) on `*.<ip>.sslip.io`. Commit: `<sha>`.

| | |
|---|---|
| `proxy boot` healthy gate | ✅ N秒 / ❌ timeout |
| UFW + sysctl 自動適用 | ✅ |
| `app status --format json` shape | ✅ `{root, expose:[{label, service}]}` |
| `app status` from /tmp (#178) | ✅ root only + stderr warning |
| HTTPS + LE prod cert (root + api) | ✅ issuer `<R3|E5|E6|E7|E8>` |
| rollback 2 経路 (default warning / --target fatal) | ✅ |
| Teardown — dangling 0 件 | ✅ servers=0, volumes=0 |

Findings: <なければ "none">

LE quota: <ip>.sslip.io 単独使用、本番 zone への影響なし。
```
