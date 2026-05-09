# ユーザー報告バグの最小再現 (issue-repro)

## 概要

`crowdy/conoha-cli` の **既報 issue を最小フローで再現してログを取る** ためのレシピ。`recipes/release-smoke.md` がリリース直前の網羅的スモーク (proxy + 多重 stack + DNS + Let's Encrypt) を狙うのに対し、こちらは「issue 本文に書かれたフロー **だけ** を fresh VPS で 1 サイクル走らせ、診断ファイルを 1 つにまとめて issue にコメントする」用途に最適化されている。

`recipes/release-smoke.md` を流すと domain / DNS / 4 SG / 8 分待機などのオーバーヘッドが乗るが、issue 検証は通常それを必要としない。逆に release-smoke にはない要素 — 「対象 issue の version 情報からどの commit で検証するか決める」「accessory 別の `docker inspect` を一括取得する」などが本レシピの中核。

## 適用タイミング

- ユーザー / 同僚から GitHub issue が上がり、まず再現可否を確認したいとき。
- `gh issue view <n>` の本文に "I ran X then Y, expected Z, got W" 系の手順がある。
- 自分側でパッチ候補がある場合は、検証 commit を切り替えて re-run することで bisect 風の確認も可能。

## 1. 対象 issue を読んで検証 commit を決める

```bash
ISSUE=195                                 # 例
REPO=crowdy/conoha-cli
gh issue view "$ISSUE" --repo "$REPO" | tee /tmp/issue-${ISSUE}.txt

# issue 本文に書かれた version (例: "v0.7.1-3-g7127ce0") を抜き出す
# その version 以降に該当領域へ入った変更を一覧
REPORTED_VER=v0.7.1                       # issue から拾う
git -C ~/dev/conoha-cli describe          # 現在のローカル HEAD
git -C ~/dev/conoha-cli log "${REPORTED_VER}..HEAD" --oneline -- internal/proxy internal/ssh internal/app
```

判断:

- 報告 version と HEAD の差が **小さい / 関連領域に変更なし** → HEAD で再現を試みる。
- 報告 version 後に **該当領域へ修正が入っている** → まず HEAD で「未再現」を確認し、次に報告 version で「再現する」を確認すれば「fixed by <commit>」と結論できる。
- 報告 version より前に類似 issue が closed されている → 重複の可能性。本レシピを実行する前に `gh issue list --state closed --search "<keywords>"` で確認。

`conoha` バイナリは `make build` で `bin/conoha` に出る。検証セッション中はこれを `PATH` 先頭に置くか、フルパスで起動する:

```bash
export PATH="$HOME/dev/conoha-cli/bin:$PATH"
conoha version    # = git describe と一致することを確認
```

## 2. 最小 fresh VPS の払い出し

issue 検証は通常 domain / DNS / Let's Encrypt が不要なので、`recipes/release-smoke.md` より軽い構成で済む。SSH と (proxy 系の issue なら) 80/443 が届けば十分。

```bash
EPOCH=$(date +%s)
SRV_NAME="repro-${ISSUE}-${EPOCH}"
WORK_DIR="/tmp/repro-${ISSUE}-${EPOCH}"
mkdir -p "$WORK_DIR"
echo "$SRV_NAME"  > "$WORK_DIR/srv-name"
echo "$WORK_DIR"  > "$WORK_DIR/work-dir"

conoha server create --no-input --yes --wait --wait-timeout 6m \
  --name "$SRV_NAME" \
  --flavor <最小プラン ID> \
  --image  <報告と同じイメージ ID> \
  --key-name <keyname> \
  --security-group default \
  --security-group IPv4v6-SSH \
  --security-group IPv4v6-Web \
  --format json | tee "$WORK_DIR/create.json"

VPS_IP=$(jq -r '.addresses // .ip // empty' "$WORK_DIR/create.json" | head -1)
echo "$VPS_IP" > "$WORK_DIR/ip"
```

ポイント:

- **イメージは報告と揃える** — `vmi-docker-29.2-ubuntu-24.04-amd64` 系で報告された UFW / sysctl 起因のバグを別イメージで検証してしまうと再現しない。
- **flavor は最小**で OK (issue が flavor 依存ならその flavor)。GPU / large memory が要件でなければ `g2l-t-c3m2` クラスで十分。
- DNS / sslip.io は **使わない**。proxy 系 issue でも `--server-name <ip>` か Host ヘッダで TLS なしの再現で足りるケースが多い。

## 3. 再現コマンド列を実行 + stderr をキャプチャ

issue 本文の手順を **そのままの順序** で走らせる。各コマンドの stdout/stderr を別ファイルに分けて保存:

```bash
run() {
  local tag=$1; shift
  echo "=== $tag === $(date -Is)" | tee -a "$WORK_DIR/run.log"
  "$@" 2> "$WORK_DIR/${tag}.err" | tee "$WORK_DIR/${tag}.out"
  echo "exit=$? for $tag" | tee -a "$WORK_DIR/run.log"
}

# 例: #195 (proxy + accessory env propagation)
run proxy-boot     conoha proxy boot     "$SRV_NAME"
run app-init       conoha app init       "$SRV_NAME" --app-name myapp --proxy
run app-env-set    conoha app env set    "$SRV_NAME" --app-name myapp KEY=value
run app-deploy     conoha app deploy     "$SRV_NAME" --app-name myapp
```

issue 本文がコマンド列でなく "behavior" を述べているだけの場合は、本文を読みながら最も近い CLI シーケンスに翻訳する。翻訳した推測コマンドは `run.log` の冒頭にコメントとして残す (再現できなかったときに「翻訳が悪かったのか / バグなのか」を切り分けられる)。

## 4. 直後の診断キャプチャ

再現を観測した直後に、**サーバー側の状態を凍結** する。再起動や次のコマンドで失われる情報があるため、ここはバッチで一気に取る:

```bash
SSH="ssh -o StrictHostKeyChecking=no root@$VPS_IP"

$SSH 'cat /opt/conoha/myapp/.env.server'                                              > "$WORK_DIR/env.server"
$SSH 'cat /opt/conoha/myapp/.conoha-mode'                                             > "$WORK_DIR/mode-marker"
$SSH 'docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"'              > "$WORK_DIR/docker-ps.txt"
$SSH 'docker compose -p myapp ps -a'                                                  > "$WORK_DIR/compose-app.txt"
$SSH 'docker compose -p myapp-accessories ps -a'                                      > "$WORK_DIR/compose-accessories.txt"

# accessory ごとに env / cmd を取る (issue 領域に応じて加減)
for c in pdns-like-container redis-like-container slot-app; do
  $SSH "docker inspect $c --format '{{range .Config.Env}}{{println .}}{{end}}'" \
    > "$WORK_DIR/inspect-env-${c}.txt" 2>/dev/null || true
  $SSH "docker inspect $c --format '{{json .Config.Cmd}}'" \
    > "$WORK_DIR/inspect-cmd-${c}.txt" 2>/dev/null || true
done

# UFW / sysctl 起因の疑いがあるとき
$SSH 'ufw status verbose; echo ---; sysctl net.ipv4.ip_unprivileged_port_start' > "$WORK_DIR/host-net.txt"
```

## 5. 判定マトリクスと issue へのフィードバック

`$WORK_DIR` の中身を眺めて結論を 1 つ選ぶ:

| 観測 | 結論 | 次のアクション |
|---|---|---|
| **再現した** & HEAD = 報告 version | 既報の通り。原因領域が明らか | issue にログを貼り、PR で fix。reproducer は `recipes/issue-repro.md` を参照したと書く |
| **再現した** & HEAD ≠ 報告 version | regression / 未修正 | issue にログを貼って "still reproduces on `$(git -C ~/dev/conoha-cli describe)`" |
| **再現しなかった** & HEAD ≠ 報告 version | 既に修正済みの可能性 | 報告 version でも実行 (build branch を切り替えて re-run §3-4)。両方の `run.log` を出して fixed-by <commit> を結論 |
| **再現しなかった** & HEAD = 報告 version | 環境依存 / 翻訳ミス / 別バグ | issue で再現条件を質問。`run.log` の翻訳コメントが効いてくる |
| **別の症状が出た** | 別 issue | 新規 issue を起票し、本検証ログを参考リンクとして記載 |

issue へのコメントは `$WORK_DIR` を 1 つの zip にまとめて添付するか、主要ログをコードブロックで貼る:

```bash
( cd "$WORK_DIR" && zip -r "/tmp/repro-${ISSUE}-${EPOCH}.zip" . )
gh issue comment "$ISSUE" --repo "$REPO" --body-file - <<EOF
Reproducer: \`recipes/issue-repro.md\` on \`$(conoha version)\`.

\`\`\`
$(cat "$WORK_DIR/run.log")
\`\`\`

Full diagnostics: \`/tmp/repro-${ISSUE}-${EPOCH}.zip\` (attached)
EOF
```

## 6. クリーンアップ

検証 VPS は **必ず** boot volume ごと落とす。`server delete` 単独だと boot volume が残って quota を圧迫する (`SKILL.md` 「サーバー管理」表参照):

```bash
conoha server delete "$SRV_NAME" --delete-boot-volume --yes --wait

# 念のため orphan が残っていないか確認
conoha volume list --no-headers | grep -F "$SRV_NAME" || echo "no orphan volume"

# WORK_DIR は issue がクローズされるまで残しておく (添付資料の根拠として)
ls -la "$WORK_DIR"
```

## 参考

- `recipes/release-smoke.md` — リリース直前の網羅スモーク。本レシピより重い。
- `SKILL.md` 「在庫切れリトライと既存ボリュームの再利用」 — 検証中の HTTP 500 回避に使える。
- `SKILL.md` 「SSH 到達性を担保する SG attach パターン」 — VPS 払い出し直後に SSH が届かないときの復旧。
