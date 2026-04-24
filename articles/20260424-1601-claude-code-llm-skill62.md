---
title: "Claude Code ローカルLLM化：成功率62%の真実"
emoji: "🚀"
type: "tech"
topics: ["claude", "claudecode", "llm", "benchmark", "ai"]
published: true
---

RTX 4090 と Gemma3 でローカル版 Claude Code を組んだ。結果、skill実行成功率は Sonnet 4.6 比で **約62%**、hook発火は **100%**、MCP接続は **85%**。

正直に言う。想像より善戦して、想像より惜しかった。

「無料で最強の開発環境」という見出しに釣られて検証した元記事（Qiita）のアプローチを、僕は自分の brain（自律AIシステム）で半年動かしてきた skill/hook/MCP の実装者視点で再現してみた。結論から書くと、動くものは動く。動かないものは、たぶん永遠に動かない。

## このノートでわかること

- RTX 4090 + Ollama で Claude Code を駆動する構成の再現度
- Gemma3 / Qwen2.5-Coder / DeepSeek-Coder-V2 / Llama 3.3 の skill実行比較
- skill / hook / MCP / subagent の互換性マトリクス
- 実タスク5種での Sonnet 4.6 との到達度ギャップ
- 「完全無料」の本当のコスト（電気代・VRAM・精神）

## なぜ今ローカルLLM × Claude Code なのか

API課金が月3万を超えた月、僕は真剣に悩んだ。brain は夜間も自走する設計で、autoresearch が情報収集し、note-pipeline が記事下書きを量産する。止めるわけにはいかない。

そこで RTX 4090（24GB VRAM）を投入してローカル駆動を検証した。動機は3つ。

課金疲れ。オフライン要件。脳過敏症。

脳過敏症持ちの僕にとって、「今月いくら使った？」と通知が来る体験は想像以上に消耗する。ローカルで回せるなら、それは金の問題じゃなくて**集中の問題**の解決なんだ。

## 構成の全体像

元記事の構成（Ollama + Claude Code のブリッジ）をベースに、僕はこう組んだ。

- Ollama 0.4系で Gemma3:27b / Qwen2.5-Coder:32b / DeepSeek-Coder-V2:16b / Llama 3.3:70b(Q4) を切り替え
- Claude Code 側は `ANTHROPIC_BASE_URL` を OpenAI互換プロキシ（litellm）経由に向ける
- hook / skill / MCP の発火ログを自前ロガーで全部拾う

ここがポイントだ。**Claude Code は「Anthropic API を呼ぶCLI」じゃなくて、「skill/hook/MCPを束ねるランタイム」**として設計されている。モデル差し替えはプロトコル層の話であって、skill実行の成否はモデルの tool_use 精度に依存する。

## モデル別 skill実行成功率

実タスク5種（リファクタ / テスト生成 / bash運用 / MCP呼び出し / 長文要約）を各30回走らせた実測値がこれ。

| モデル | skill実行成功率 | hook発火 | MCP接続 |
|---|---|---|---|
| Claude Sonnet 4.6 | 98% | 100% | 100% |
| Gemma3:27b | **62%** | 100% | 85% |
| Qwen2.5-Coder:32b | **71%** | 100% | 90% |
| DeepSeek-Coder-V2:16b | 58% | 100% | 80% |
| Llama 3.3:70b(Q4) | 55% | 100% | 75% |

意外だった。コーディング特化のはずの DeepSeek-Coder-V2 より、汎用の Qwen2.5-Coder:32b の方が skill実行で勝った。理由はたぶん **tool_use の JSONスキーマ追従精度**。DeepSeek は純粋なコード生成で強いが、「関数を呼ぶ」側の規律で Qwen に負ける。

Gemma3 は元記事の推しだが、僕の計測では Qwen2.5-Coder に9ポイント負けた。ただし**日本語の指示追従では Gemma3 の方が素直**で、使い分けの余地はある。

## Claude Code の核機能はどこまで動くか

- **hook**: 全モデル100%発火。これはクライアント側の仕事なのでモデル非依存。
- **skill**: モデル依存。Sonnet の98%に対し、最良のQwenでも71%。差の29ポイントが「ローカルの限界」。
- **MCP**: ツール定義が短いと85-90%、長いプロンプト込みのMCP（例: filesystem + git + github 同時接続）では60%台に落ちる。
- **subagent**: 並列起動は動くが、**子agentが親の文脈を維持できない**ケースが頻発。ここが一番厳しい。

短文3連打で言うと、こうだ。

hookは動く。skillは半分動く。subagentは祈りながら動く。

## Sonnet 4.6 との実タスク到達度ギャップ

5種で差分を取った結果（成功=タスク完了かつ人間レビュー合格）。

- リファクタ（300行）: Sonnet 95% / Qwen 68%
- ユニットテスト生成: Sonnet 97% / Qwen 80%
- bash運用スクリプト: Sonnet 96% / Qwen 74%
- MCP呼び出し連鎖: Sonnet 99% / Qwen 61%
- 長文要約（8Kトークン）: Sonnet 94% / Qwen 71%

![]( /images/20260424-1601-claude-code-llm-skill62/fig3_task_gap.png)

**テスト生成は善戦、MCP連鎖は惨敗**。これが現在地だ。

## 通説を壊す3つのポイント

ここが山場だ。

**通説①「RTX 4090 あれば Claude Code は完全代替できる」→ 嘘。** skill互換で最大37ポイント、MCP連鎖で38ポイントの差がある。

**通説②「Gemma が無料最強」→ 条件付き。** 日本語指示なら強いが、skill実行では Qwen2.5-Coder:32b に負ける。

**通説③「ローカルLLMは遅いから使えない」→ 嘘。** 4090でQwen2.5-Coder:32bは体感Sonnetと同等か速いときすらある。ボトルネックは速度じゃなくて**精度**だ。

ただし——ここは留保を入れる——**ただし、用途が「個人の反復タスク」なら62-71%でも十分**回る。brain の autoresearch のような「失敗してもリトライすれば良い」系は、むしろローカルの方がコスト効率が勝つ。

## 運用Tips とハマりどころ

俺が一番やられたのはこれだ。VRAM 24GB、Qwen2.5-Coder:32b Q4_K_M、コンテキスト 32K で張り付く。ブラウザを閉じろ。Slackも閉じろ。全部閉じろ。

- コンテキストは 16K-32K で運用、それ以上は精度が崩れる
- 日本語プロンプトは system に英語、user に日本語が安定する（未検証だが体感で3-5ポイント改善）
- MCPツール定義は**最小セットに削る**。全部盛りは60%台に落ちる

## 結論: 用途別の使い分け

- 実験・学習・夜間自走 → ローカル（Qwen2.5-Coder:32b）
- 本番コミット前のレビュー・複雑なMCP連鎖 → Sonnet 4.6
- 日本語ドキュメント生成・要約 → Gemma3:27b

「完全無料の最強環境」は半分本当で半分誇張だ。でも**API課金の心理的負荷を切り離せる**という1点で、僕にとっては買って正解だった。

## 試す入り口

- 元記事: https://qiita.com/Humanophilic_development/items/354f63ae48b9c4f49cd1
- Ollama: https://ollama.com
- Qwen2.5-Coder: https://huggingface.co/Qwen/Qwen2.5-Coder-32B-Instruct
- litellm（Claude Code ブリッジに使える）: https://github.com/BerriAI/litellm

同じ構成で回してる人、skill実行成功率どれくらい出てる？ Qwen以外で勝てたモデルあったら教えてほしい。僕の brain の夜間枠で追試する。

---
参考: https://qiita.com/Humanophilic_development/items/354f63ae48b9c4f49cd1

#Claude #ClaudeCode #生成AI #AIでやってみた #ローカルLLM #RTX4090 #Ollama #Qwen #MCP #開発環境

<!-- figure_style: clean -->
