# yawara

[English](README.md) | 日本語

Claude Code の返答と `AskUserQuestion` の質問文・選択肢を、Antigravity CLI (`agy`) + Gemini 3.8 Flash (high) で「人間にとってわかりやすい文章」に書き換えて表示する Hooks です。[APM (Agent Package Manager)](https://github.com/microsoft/apm) のパッケージ形式で配布します。

書き換え前と後の文章は、すべて `~/.yawara/logs/` に記録されます。

## 必要なもの

- Claude Code (`MessageDisplay` フックに対応した版。v2.1.310 で動作確認済み)
- APM CLI (`apm`。v0.29.0 で動作確認済み)
- Antigravity CLI (`agy`。v1.4.7 で動作確認済み) と、ログイン済みの状態
  - フックは `agy -p` (ヘッドレスモード) で呼び出します。事前に一度 `agy` を対話モードで起動してログインしておいてください。
- `python3` (3.9 以上。標準ライブラリのみ使用)

## インストール

すべての Claude Code セッションで使う場合 (ユーザー全体、`~/.claude/settings.json` に登録):

```bash
apm install -g asuka1975/yawara#v0.1.1
```

特定のプロジェクトだけで使う場合 (そのプロジェクトの `.claude/settings.json` に登録):

```bash
cd your-project
apm install asuka1975/yawara#v0.1.1
```

インストール後に起動した Claude Code セッションから有効になります。外すときは `apm uninstall asuka1975/yawara` (グローバルなら `-g` 付き) を実行します。`apm uninstall yawara` のように名前だけでは指定できません。

## 何をするか

| 対象 | 使うフック | 動き |
| --- | --- | --- |
| Claude の返答メッセージ | `MessageDisplay` | ストリーミング中の行はいったん表示せずに貯めておき、メッセージが最後まで届いた時点で全文をまとめて書き換えて表示します。 |
| `AskUserQuestion` の質問 | `PreToolUse` (matcher: `AskUserQuestion`) | 質問文 (`question`)、選択肢の名前 (`label`)、選択肢の説明 (`description`) を 1 回の呼び出しでまとめて書き換え、`updatedInput` で差し替えます。`header` とプレビューはそのままです。 |

書き換えのルール (プロンプトは `.apm/hooks/scripts/yawara.py` 内):

- 元と同じ言語で書く。意味・事実・結論は変えず、情報を削ったり足したりしない。
- 結論を先に置き、長い文は分ける。専門用語は残したうえで、かっこ書きで短い言い換えを添える。
- コード、コマンド、ファイルパス、URL、数値、識別子、エラーメッセージは一字一句そのまま。

### 知っておいてほしいこと

- **表示が遅くなります。** 書き換え 1 回にかかる時間は、実測で 10〜70 秒程度でした。その間、メッセージは表示されません (スピナーに「yawara: rewriting the message for readability…」と出ます)。ターンが「done」になった後に、書き換え済みのメッセージが遅れて表示されることもあります。
- **書き換わるのは画面表示だけです。** トランスクリプトや Claude 自身が見る文章は元のままです。`ctrl+o` の詳細表示でも元の文章が見えます。
- **`AskUserQuestion` の回答は、書き換え後の文言で Claude に返ります。** そのため、元の文言との対応表を `additionalContext` で Claude に渡しています。
- **失敗しても止まりません。** `agy` がエラー・タイムアウトになったときや、出力が壊れていたときは元の文章をそのまま表示し、ログに `"status": "fallback"` と理由を残します。100 KB を超える長いメッセージは書き換えません (`agy` にはプロンプトをコマンド引数で渡すため)。
- **表示される文章は Google (Gemini) に送信されます。** コードやファイルパスなど、Claude の返答に含まれる内容がそのまま `agy` に渡ります。外部に出せない情報を扱うプロジェクトでは、プロジェクト単位のインストールにするか、`YAWARA_DISABLE=1` で止めてください。

## ログ

`~/.yawara/` 以下は、フックの初回実行時に自動で作られます。

```text
~/.yawara/
├── logs/YYYY-MM-DD.jsonl   # 変換前後のログ (1 行 1 件、日付ごとのファイル)
├── state/                  # ストリーミング中のメッセージを一時的に貯める場所 (表示後に削除)
└── workspace/              # agy を実行する作業ディレクトリ (プロジェクトの文脈を読み込ませないため)
```

ログ 1 件の形式:

```json
{
  "timestamp": "2026-10-26T02:23:12+09:00",
  "kind": "message",
  "status": "rewritten",
  "error": null,
  "model": "gemini-3.8-flash-high",
  "duration_sec": 24.8,
  "session_id": "…",
  "cwd": "/path/to/project",
  "message_id": "…",
  "before": "元の文章",
  "after": "書き換え後の文章"
}
```

- `kind`: `message` (返答) または `ask_user_question` (質問)。`ask_user_question` では `message_id` の代わりに `tool_use_id` が入り、`before` / `after` は `questions` 配列になります。
- `status`: `rewritten` (書き換えた) または `fallback` (失敗したので元の文章を表示した。`after` は `null`)。

例: 今日のログを読みやすく表示する

```bash
jq -r '"[\(.timestamp)] \(.kind) \(.status)\n--- before\n\(.before)\n--- after\n\(.after)\n"' ~/.yawara/logs/$(date +%F).jsonl
```

## 設定

環境変数で変更できます。Claude Code に渡すには、`~/.claude/settings.json` の `env` に書くか、Claude Code を起動するシェルで設定してください。

| 環境変数 | 既定値 | 内容 |
| --- | --- | --- |
| `YAWARA_MODEL` | `gemini-3.8-flash-high` | `agy` に渡すモデル。使えるモデルは `agy models` で確認できます。 |
| `YAWARA_TIMEOUT` | `150` | `agy` 1 回あたりの制限時間 (秒)。フック自体の `timeout` (180 秒) より短くしてください。長くすると、フックが先に打ち切られ、元の文章の一部しか表示されなくなります。 |
| `YAWARA_HOME` | `~/.yawara` | ログ・一時ファイルの置き場所。 |
| `YAWARA_AGY` | `agy` | `agy` コマンドのパス。デスクトップアプリから起動した Claude Code などで `PATH` に `agy` が無いと、ログがすべて `FileNotFoundError` の `fallback` になります。その場合は `which agy` の結果 (絶対パス) を設定してください。 |
| `YAWARA_DISABLE` | (なし) | `1` にすると書き換えをせず、元の文章をそのまま表示します。 |

例 (`~/.claude/settings.json`):

```json
{
  "env": {
    "YAWARA_MODEL": "gemini-3.8-flash-medium"
  }
}
```

## パッケージの構成

```text
yawara/
├── apm.yml                      # APM マニフェスト (targets: claude)
└── .apm/hooks/
    ├── yawara.json              # フック定義 (MessageDisplay / PreToolUse)
    └── scripts/yawara.py        # 書き換え・ログ記録の本体
```

`apm install` すると、フック定義は `.claude/settings.json` (グローバルなら `~/.claude/settings.json`) にマージされ、スクリプトは `.claude/hooks/yawara/scripts/yawara.py` に配置されます。

## 動作確認

フックは標準入力で JSON を受け取るので、Claude Code を通さずに試せます。

```bash
echo '{"hook_event_name":"MessageDisplay","message_id":"test","index":0,"final":true,"delta":"PreToolUse フックで updatedInput を返すとツール入力を差し替えられます。\n"}' \
  | YAWARA_HOME=/tmp/yawara-test python3 .apm/hooks/scripts/yawara.py
```

## ライセンス

[GNU General Public License v3.0 or later](LICENSE) (`GPL-3.0-or-later`)
