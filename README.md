# claude-code-skills

Tsuji Mitsunari's shared Claude Code skills.

## Skills

### `usage_snapshot`

Claude Code 利用ログを「N=1スナップショット」として採取するエージェントスキル。
業務トピックの代表的サンプル原文＋構築物サマリを1ファイルにまとめる。

- 質問しない（完全非同期）
- 要約しない・1文字も書き換えない
- 業務語彙はユーザ本人の履歴から自己学習（固定KWを押し付けない）

→ [`skills/usage_snapshot/SKILL.md`](skills/usage_snapshot/SKILL.md)

## 使い方

### 方法1：raw URL から直接 WebFetch（Claude Code 内）

```
以下の SKILL.md を読んで、その指示に従って実行してください。

https://raw.githubusercontent.com/tsuji-ai/claude-code-skills/main/skills/usage_snapshot/SKILL.md
```

### 方法2：clone してローカル参照

```bash
git clone https://github.com/tsuji-ai/claude-code-skills.git ~/claude-code-skills
```

Claude Code に：
```
~/claude-code-skills/skills/usage_snapshot/SKILL.md を読んで実行
```

## ライセンス

個別配布。汎用利用可。
