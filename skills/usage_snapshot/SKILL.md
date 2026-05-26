---
name: usage_snapshot
description: あなた（=今このスキルを実行している人）の Claude Code 利用ログを「N=1スナップショット」として採取し、指定された Google Drive フォルダに格納するエージェントスキル。業務トピックの代表的サンプル原文＋構築物サマリを1ファイルにまとめる。配布時の目的・出力先・期限などはチャットプロンプト側で個別指定される前提。質問せず・要約せず・1文字も書き換えず・捏造せず採取し、書き出して Drive 格納まで完走する。
---

# usage_snapshot — Claude Code 利用ログ N=1 スナップショット採取エージェント

## ゴール

あなた（=このスキルを実行しているユーザ）の Claude Code 過去ログから、
**業務トピックでの使い方** の N=1 サンプルを採取し、
`usage_snapshot_<ユーザ名>_<日付>.md` として書き出し、
**指定された Google Drive フォルダに格納するまで自走** する。

「業務トピックでの使い方」とは、ユーザ本人の語彙で
「自分の仕事に関係のあるやり取り」と判別できるものを指す。

このスキル自体は **目的に依存しない**。
「何のために採取するか／どこに置くか／期限」は配布時のチャットプロンプトに書かれているはず。
書かれていなければ、本スキルのデフォルト（ローカル `~/Desktop/`）で進めて手順を提示する。

## 絶対制約（順守必須）

1. **ユーザに質問しない**（完全非同期）
2. **要約しない・分類しない・「気づき」「示唆」「まとめ」を書かない**
3. **ユーザの発話は1文字も書き換えない**（言い淀み・誤字・口ぐせを保存）
4. **推定箇所すべてに `[推測]` マーカー**
5. **該当なし・取得失敗は素直に「該当なし」と記録**（捏造禁止）
6. 書き出し→Drive格納まで完走したら終了。確認打診なし。

## 自由度（あなたが判断していい部分）

- **業務トピックの判定方法** ：ユーザのmanualセッションを20件ほど読んで、ユーザ本人が頻繁に使う「業務語彙」を学習。固定キーワードを押し付けずに、本人の語彙で判定基準を作る
- **母集団サイズ別のサンプリング戦略** ：母集団が小さければ全件、大きければ多様性を担保した抽出
- **出力ファイルの細部構造** ：最低限の構造（下記）は守るが、見出しレベル・順序・補助セクションは任意

## 推奨フロー

### Step 1: ユーザ名の特定（質問なし）

優先順位：
1. `git config user.name`
2. `~/.claude.json` の userName フィールド
3. `~/.gitconfig` の user.name
4. それも無ければ `unknown_user`

### Step 2: 履歴全件スキャン

- `~/.claude/projects/` 配下の `.jsonl` を全件列挙（worktree含む）
- 各セッションについて：path / timestamp / size / 起動パターン / 往復回数
- 起動パターン分類：
  - `manual`：普通の文で始まる（ユーザが手で書いた）
  - `skill_dispatch`：「Base directory for this skill」で始まる
  - `launchd_auto`：「【launchd 自動実行】」で始まる

### Step 3: 業務語彙の自己学習

manual セッションのうち、最近20件ほどの最初のuserメッセージを読んで、
**ユーザ本人が頻繁に使う名詞・動詞** を抽出。

ここで作る「業務語彙リスト」は **ユーザ固有** であって、辻実成や他人の語彙を流用しない。
（例：ある人は「お客さん」、別の人は「クライアント」、別の人は「顧客」。本人の発話に従う）

### Step 4: 業務トピックフィルタ

manualセッションの最初のuserメッセージに、Step 3 で学習した語彙が含まれるものを抽出。
**往復回数2以上**のみ採用（試打除外）。

このフィルタを通った母集団を M とする。

### Step 5: サンプリング（母集団サイズで分岐）

- **|M| < 2件**：「業務トピックの manual セッションなし」と素直に記録（捏造禁止）
- **2 ≤ |M| < 20件**：全件採用
- **|M| ≥ 20件**：往復回数で sort し、
  - 中央値5件（メディアン±2）
  - 外れ値5件（深い対話 top3 ＋ 短い対話 bottom2）
  - 合計10件採用

各サンプルは以下をそのまま記録：
- timestamp / size / 往復回数 / path
- 最初のuserメッセージ全文（途中で切らない）
- 最後のassistantメッセージ末尾30行

### Step 6: 構築物（取れたものだけ）

質問せず取得を試行：
- `~/.claude/skills/` または `_agent/skills/` 配下の `ls` 結果（あれば）
- `~/Library/LaunchAgents/` の plist `ls`（あれば）
- カレント／ホーム／`~/.claude/` 配下のCLAUDE.md冒頭30行（あれば）
- 同じく settings.json 冒頭30行（あれば）

取れなければ「該当なし」。

### Step 7: ローカル書き出し

ファイル名：`usage_snapshot_<ユーザ名>_<YYYY-MM-DD>.md`
ローカル一時置き場：`~/Desktop/<ファイル名>` または配布プロンプトで指定された絶対パス

最低限の構造：

```markdown
---
date: YYYY-MM-DD
user: ユーザ名
total_sessions: XX
manual_sessions: XX
skill_dispatch_sessions: XX
launchd_auto_sessions: XX
business_topic_pool: XX
sampled: XX
sampling_mode: all / 10samples_median5+outlier5 / none
generated_by: usage_snapshot skill
---

# 0. 定量データ
- 起動パターン件数
- 業務トピック母集団サイズ
- 往復回数分布（min/p25/median/p75/max）
- 直近10セッションのタイムライン

# 1. 業務トピックの manual セッション サンプル
- 各サンプル：ラベル（中央値／外れ値）／往復回数／原文／末尾30行

# 2. 構築物サマリ
- スキル一覧／plist／CLAUDE.md／settings.json（取れたら）

# 3. 採取できなかったもの
- 失敗パス・理由
```

### Step 8: Google Drive 格納（3段階フォールバック）

配布プロンプトで **Drive folder URL / folder ID** が指定されている前提。
以下の優先順位で1つでも成功したら終了：

**優先1：Drive MCP が使える場合**

利用可能ツールの中に以下のような MCP ツールがあるか確認：
- `*create_file*` で Google Drive 系
- `*upload*` で Drive / Google 系
- `mcp__*` 名前空間に Drive 関連ツール

見つかったら、folder ID 配下にファイルをアップロード。

**優先2：`~/.google_api_token.json` ＋ Python ／ Drive REST API**

トークンファイルが存在するか確認：
```bash
ls -la ~/.google_api_token.json 2>/dev/null
```

存在すれば、`curl` / `python3` で Drive API `files.create` を直接叩く：
```python
import json, requests
tok = json.load(open(os.path.expanduser('~/.google_api_token.json')))['token']
metadata = {'name': '<ファイル名>', 'parents': ['<folder_id>']}
files = {
    'metadata': ('metadata.json', json.dumps(metadata), 'application/json'),
    'file': ('<ローカルパス>', open('<ローカルパス>','rb'), 'text/markdown')
}
r = requests.post(
    'https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart',
    headers={'Authorization': f'Bearer {tok}'},
    files=files
)
print(r.status_code, r.json())
```

もしくは Vault に同梱の `gapi.py` を呼ぶ：
```bash
ls "/Users/mt/myRAG/Obsidian Vault/90_Built/gas/gapi.py" 2>/dev/null
```
が存在すれば、それを使って upload する。

**優先3：手動アップロード手順を画面に表示して終了**

優先1・2どちらも使えなければ、ローカルファイルを残したまま、ユーザに：
- ローカルファイルの絶対パス
- Drive folder URL
- 「このパスのファイルを上記フォルダに手動でドラッグ＆ドロップしてください」

を画面に表示して終了。**自分でユーザに質問はしない**。

## 「業務語彙の自己学習」の注意点

このスキルが既存（他の人の）成功例を学習して語彙を流用するのは禁止。
**必ず実行中ユーザのmanualセッションだけ** から語彙を抽出する。

例：辻実成の業務語彙には「金剛 / MOCA / 接客 / クチコミ」が含まれるが、
別の人のスナップショットを取る時に、これらを **押し付けてはいけない**。
別の人は「クリニック / 患者 / 受付」かもしれないし、「現場 / 工事 / 職人」かもしれない。

## このスキルが想定する典型ユースケース

- 組織で Claude Code を展開する責任者が、既に触っている数名から「先行している人の使い方」を取りたい
- 1ヶ月後／3ヶ月後に同じスキルを再実行して **before → after の対比基準** を作りたい
- 説得や評価のためではなく、**「自分にも合いそう」「ここからなら踏み出せる」と感じられる呼び水** にしたい

配布プロンプトには、上記のような **採取の目的・読み手・使われ方** が書かれている前提で動く。
書かれていれば守る。書かれていなければデフォルト動作で完走する。
