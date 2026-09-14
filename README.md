# luna-shunt

大きなファイルの読み込みと定型コードの生成を `gpt-5.6-luna` のサブエージェントへ
委譲するよう Codex に促す Codex CLI プラグインである。

[haiku-shunt](https://github.com/macha434/haiku-shunt)(Claude Code版)の
Codex 移植だが、**仕組みが同じではない**。違いは下の「Claude Code版との違い」
を必ず読むこと。

## 概要

- 対象ユーザー: 大きなファイルの読み込みと機械的なコード生成がトークン消費の
  大半を占めている Codex CLI ユーザー
- 解決する課題: メインの会話で 2,000 行のファイルを読むと、その内容はスレッド
  が終わるまでコンテキストに残り続け、以降のすべてのターンで課金対象になる
- 価値: 同じファイルを子エージェントに読ませれば、コストは要約 1 回分で済む。
  会話に届くのは答えだけで、ファイル本体は破棄される

Spotify が Claude Code のトークン使用量を約 90% 削減した手法(大きい読み込みと
定型生成を安いモデルへ委譲する)を、Codex CLI 向けに再構成したものである。

## Claude Code版との違い(重要)

haiku-shunt は PreToolUse フックで大きいファイルの直接読み込みを**強制的に
ブロック**する。Codex のプラグイン体系にはツール呼び出しを横取り・ブロックする
仕組みが存在しない(`plugin.json` の `hooks` フィールドはスキーマ上は存在する
ものの、実際のバリデーションでは拒否される)。

そのため luna-shunt は**強制ではなく、スキルによる運用ルール**として実装して
いる。Codex がこのスキルを毎回忠実に実行する保証はない。それでも「委譲すれば
安い」という判断基準を Codex に持たせておくこと自体に価値があると判断した。

## 主な機能

- **`delegating-bulk-work` スキル**: 大きいファイルの読み込みや、設計の決まった
  コード生成を `spawn_agent` で `gpt-5.6-luna` の子エージェントへ委譲する具体的な
  手順(呼び出し形、プロンプトの書き方、落とし穴)を Codex に指示する

## 必要なもの

- プラグインに対応した Codex CLI(`multi_agent`/`multi_agent_v2` が有効であること。
  多くのビルドで既定で有効)
- `spawn_agent` のツールスキーマに `model`/`reasoning_effort` フィールドが
  含まれていること。含まれない場合はスキル内に記載した `config.toml` の
  ワークアラウンドを試す(未文書化のため効かないビルドもありうる)

## セットアップ

```bash
codex plugin marketplace add macha434/luna-shunt
codex plugin add luna-shunt@macha434-plugins
```

インストール後、新しいスレッドで試すこと(既存スレッドはプラグイン追加後の
スキルを拾わない)。

## 使い方

呼び出す操作は不要である。スキルの説明文が「数百行を超えるファイルを読む前」
「ひな型やテストを書く前」に一致する場面で、Codex が自律的にスキルを読み、
委譲するかどうかを判断する。

明示的に促すこともできる。

> 大きいファイルの調査は luna-shunt で委譲して。`src/http/` のどこでリトライ
> ポリシーが設定されているか調べて。

## 既知の制約

- **強制力がない** — haiku-shunt のフックのような技術的な強制ブロックはない。
  Codex がスキルの指示を無視すれば、そのまま親のモデルで読み込む
- **`spawn_agent` の `model` 指定が非公式** — `hide_spawn_agent_metadata` は
  OpenAI公式ドキュメントに未記載のワークアラウンドであり、ビルドによって
  効かない、または将来のアップデートで変わる可能性がある
- **`model` の allowlist** — `multi_agent_v2` は「V2対応のプリセットのみ受理し、
  それ以外はハードエラーになる」という報告がある。`gpt-5.6-luna` が使っている
  Codex ビルドの spawn allowlist に含まれているか、事前に確認すること

## クレジット

委譲によるコスト最適化のアイデアは
[spotify/portal-ai-plugins](https://github.com/spotify/portal-ai-plugins)
(Claude Code 向け)、および Codex 版としての再構成は
[haiku-shunt](https://github.com/macha434/haiku-shunt) に由来する。

## ライセンス

MIT
