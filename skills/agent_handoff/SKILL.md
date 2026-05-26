---
name: agent_handoff
description: 既存の汎用スキル/プロンプトを「他者に委譲して走らせる」ための型化スキル。①個別目的・出力先・期限を載せた dispatch_prompt を作成 → ②GitHub repo に publish して raw URL 確保 → ③配布メッセージドラフトを message_approval_gate 経由で起案 → ④handoff履歴を _agent/handoff_log.md に追記。arch_jirita（自分は動かずに望むものを得る）の物理化。配布対象＝知り合い・社内同僚・クライアント。
type: orchestrator
trigger_phrases:
  - handoff
  - 委譲して
  - 配布したい
  - 共有指示メッセージ
  - 別の人に走らせて
  - dispatch_prompt
related_skills:
  - arch_jirita
  - message_approval_gate
  - message_factory
related_rules:
  - .agent/rules/10_message_approval.md
  - .agent/rules/11_storage_tiering.md
---

# agent_handoff — エージェント委譲キット

> 既存の汎用スキル（例 `usage_snapshot`）を、特定の目的・宛先・期限を載せて
> 第三者（社内同僚／クライアント／知人）の Claude Code に走らせる一連の型。
>
> 単発の dispatch メッセージを毎回手で組み立てるのではなく、
> **dispatch_prompt 生成 → GitHub publish → 配布メッセージ起案 → 履歴追記** までを
> 1スキルで物理化する。

---

## なぜこのスキルか

辻メソッドの中核：
- 装置を組んで自分は動かない（[arch_jirita]）
- 既にワークしているもの（汎用スキル）を、それぞれの相手に合わせた包みで渡す
- 「同じ装置を10人に手で配る」のではなく「配布の段取り自体を装置化する」

CLAUDE.md 原則との整合：
- **SSoT原則**：原本は Vault `_agent/`、GitHub は publish 先（一方向ミラー）
- **Antigravity共生**：両AIが Vault 原本を読める
- **rule 10**：配布メッセージは必ず `message_approval_gate` 経由
- **rule 11**：publish 前に個人情報含有チェック

---

## 入力（必須5＋任意）

| 項目 | 必須 | 内容 |
|:--|:--:|:--|
| `skill_name` | ✅ | 委譲する汎用スキル名（例 `usage_snapshot`）。raw URL も自動解決 |
| `purpose` | ✅ | 今回の目的・背景・読み手・使われ方（文章） |
| `output_dest` | ✅ | 出力先（例 Google Drive folder URL ／ folder ID） |
| `deadline` | ✅ | 期限（例 `2026-06-04`） |
| `recipients` | ✅ | 配布先（人名＋媒体。例「土井店長・中村副店長 / Cybozu AI導入設計室」） |
| `project_path` | 任意 | プロジェクト文脈の置き場（例 `40_Projects/.../05_2026-06-11_kicho_kaigi/`）。未指定なら `_agent/handoff/<date>_<topic>/` |
| `topic_slug` | 任意 | dispatch_prompt のファイル名（例 `2026-06-11_kicho_kaigi`）。未指定なら自動命名 |
| `github_repo` | 任意 | publish 先（既定 `tsuji-ai/claude-code-skills`） |
| `mode` | 任意 | `--draft-only` / `--no-publish` / 全実行（既定） |

---

## Phase 0：原典・既存成果物の確認（arch_jirita Step 0 連携）

ドラフト生成の **前** に必ず通す：

1. プロジェクト文脈の Read（`project_path` 配下の `00_index.md` / `*requirements*` / `CLAUDE.md` 等）
2. 「今回の事例がなんのために要るか」を一次資料から把握する
3. 既存の関連成果物（同テーマで過去に作った dispatch_prompt 等）の有無を確認
4. 配布相手の人物プロファイル（[reference_optkongo_people] 等）を確認

**Why**：[feedback_my_draft_habits_observed]「読まずに書く・知らずに書く」事故12項の物理予防。

---

## Phase 1：dispatch_prompt の生成（一次保存：プロジェクト SSoT）

### 中身の必須セクション

1. **このタスクの背景**（読み手＝Claude が文脈を持って実行できるよう書く）
   - なぜ採取するか
   - 役割（説得用ではない／呼び水／比較基準 等）
   - そのまま見せる予定の有無
2. **起動**：スキル raw URL の WebFetch 指示
3. **格納先**：Drive folder URL / folder ID（必達）
4. **期限**
5. **完了時の出力フォーマット**（短く・3〜5行・要約させない）
6. **重要：途中で質問しないこと**（完全非同期）

### 保存先（2箇所）

- 一次SSoT：`<project_path>/dispatch_prompt_<topic_slug>.md`
- publish 用：`/tmp/agent_handoff_publish/<topic_slug>.md`（Phase 2で repo に同期）

---

## Phase 2：GitHub publish（一方向ミラー・`--no-publish` 時はスキップ）

### 個人情報含有チェック（rule 11 連携）

publish 前に以下を機械チェック：
- 個人メールアドレス（@gmail.com / @icloud.com 等）
- 個人電話番号
- 自宅住所
- 顧客固有の機微情報
- 検出されたら **停止** して辻に確認を仰ぐ

### push 手順

```bash
# 1. ローカル clone（既存ならpull）
REPO_DIR="${HOME}/Library/Caches/agent_handoff/claude-code-skills"
if [ ! -d "$REPO_DIR" ]; then
  gh repo clone tsuji-ai/claude-code-skills "$REPO_DIR"
else
  git -C "$REPO_DIR" pull --rebase
fi

# 2. dispatch_prompt をコピー
cp "/tmp/agent_handoff_publish/<topic_slug>.md" \
   "$REPO_DIR/dispatch_prompts/<topic_slug>.md"

# 3. commit & push
cd "$REPO_DIR"
git add "dispatch_prompts/<topic_slug>.md"
git -c user.email=optkongo2015@gmail.com -c user.name="Tsuji Mitsunari" \
    commit -m "dispatch_prompts/<topic_slug>.md 追加"
git push

# 4. raw URL を確定
echo "https://raw.githubusercontent.com/tsuji-ai/claude-code-skills/main/dispatch_prompts/<topic_slug>.md"
```

---

## Phase 3：配布メッセージ起案（message_approval_gate 経由）

### 配布メッセージの型（軽め・北原節寄り）

```
<相手呼びかけ>

<1〜2行で目的＞ <スキル名> を Claude Code で1回走らせてもらえますか。
所要 数分・本人作業ゼロ（貼って放置でOK）です。

▼プロンプト本体
<raw URL>

上記URLを開いて、表示される全文をコピーして Claude Code に貼り付けてください。
あとは Claude が <出力先の中身> に上げて終わります。
質問は飛んできません。

期限：<deadline>
不明点あれば気軽に。
```

### 承認ゲート発火

`message_approval_gate` の SKILL.md を読んでプロトコルに従う。
特に：
- Phase 0 OS_Index 物理検証
- Phase 0.7 arch_cooling Layer 判定
- destination / send_method 確定
- `message_approval_record.py pending` で投稿

---

## Phase 4：handoff_log 追記

`_agent/handoff_log.md` に1行追記（CSVライク・タブ区切り）：

```
<date>	<skill_name>	<topic_slug>	<recipients>	<raw_url>	<approval_title_ts>	<status>
```

- date：今日
- skill_name：例 `usage_snapshot`
- topic_slug：例 `2026-06-11_kicho_kaigi`
- recipients：例 `土井店長,中村副店長`
- raw_url：Phase 2 で確定した URL
- approval_title_ts：Phase 3 の Slack 承認 ts
- status：`approval_pending` ／ `--draft-only` ／ `--no-publish`

→ 後から「誰に何を渡したか／どのスキルを何回回したか」が追跡可能。

---

## 段階制御モード

| モード | Phase 0 | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|:--|:--:|:--:|:--:|:--:|:--:|
| `--draft-only` | ✅ | ✅ | ❌ | ❌ | ✅（status=draft） |
| `--no-publish` | ✅ | ✅ | ❌ | ✅ | ✅（status=no-publish） |
| 既定（全実行） | ✅ | ✅ | ✅ | ✅ | ✅ |

辻が中身を見てから進めたい場合は `--draft-only` でまず dispatch_prompt を読み、
納得したら追加実行で `--no-publish` or 既定モードへ。

---

## 終了時の報告（辻への一行）

```
✅ agent_handoff 完了
  skill: <skill_name>
  topic: <topic_slug>
  raw URL: <raw_url>
  Slack承認: title_ts=<title_ts>
  log: _agent/handoff_log.md に追記済
```

👍 で配布メッセージが自動送信される（既定モード時）。

---

## 想定ユースケース

- 社内同僚に汎用スキルを走らせて素材を集める（usage_snapshot のような採取系）
- クライアントに「これ走らせてください」を渡す（ノック1件の入口）
- 6/11店長会議の土井・中村への usage_snapshot 配布が **このスキル誕生の起点**

---

## 設計思想（CLAUDE.md整合の確認）

| 原則 | 守り方 |
|:--|:--|
| SSoT原則 | 原本は Vault `_agent/skills/agent_handoff/SKILL.md`。GitHub は publish ミラー |
| Antigravity共生 | 両AIが Vault 原本を読む |
| rule 10 message_approval | Phase 3 は必ず `message_approval_gate` を上位 orchestrator として呼出 |
| rule 11 storage_tiering | Phase 2 publish 前に個人情報チェック（rule 11 の C層基準） |
| skill marketplace sync | `skills_marketplace_sync.py` で `/agent-handoff` 起動可 |
| 読まずに書く防止 | Phase 0 で原典・既存成果物を物理確認 |
| 完全非同期 | Phase 3 後は辻の👍待ち。Claude セッション閉じてOK |
