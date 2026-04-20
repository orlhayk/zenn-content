---
title: "Claude Opus代が1/30に削減。ローカルQwenをsubagentに繋いだだけ"
emoji: "💰"
type: "tech"
topics: ["claude", "llm", "ai", "agent", "optimization"]
published: true
---

夜間バッチ1回で$3.60だったOpus課金が、$0.12に落ちた。Opus 4は今も毎晩、同じ本数だけ回している。止めていない。削ったのは「Opusに投げる必要がなかったタスク」だけだ。

月$108が$3.60になった計算になる。正直に言う、魔法ではない。Orchestrationだけ残して、他を全部ローカルに逃がしただけだ。

---

## このノートでわかること

- Claude Code subagentにLM Studio経由のローカルモデルを繋ぐ具体設定
- どのタスクをローカルに落とし、どれをOpusに残すかのルーティング設計
- 実測のトークン削減と、品質を落とさないための線引き
- 課金不安ゼロで自律システムを回すための構成判断

---

## Opusのトークン代が怖くてブレーキを踏んでいた

brainという自作の自律AIシステムを夜間に自走させている。でもOpusに重タスクを投げるたびに、カウンターが回る感覚があった。

要約。下書き。ログ整形。

全部Opusに任せていたのが問題だった。autoresearchでRedditやHacker Newsから収集して、記事を分析して、skill/hookで自動実行する。これを全部Opus経由で回すと1夜で$3.60。毎日で月$108。個人の可処分所得でやる規模じゃない。

それで考えた。**Opusじゃなくていいタスクを、Opusに投げるな。**

---

## 仕組み：Claude Code subagentとローカルLLMをつなぐ

Claude Codeにはsubagentがある。`~/.claude/agents/`にMarkdownを置くだけで、メインのOrchestrator（Claude）が必要なタイミングで専門エージェントを呼び出す。

効くのは、**subagentに任意のモデルを指定できる**点だ。

```markdown
---
name: local-worker
model: qwen3-30b-a3b
---
```

LM StudioはローカルでOpenAI互換APIを立てられる。Claude Codeの設定でsubagent用にそのエンドポイントを向ければ、Orchestratorはクラウド（Opus）、ワーカーはローカル（Qwen3）という分業が成立する。

構成はこうなる。

```
Claude Opus（Orchestrator）
  ├── 判断・計画・最終チェック → Opus API
  ├── コード生成・要約・分析 → Qwen3 via LM Studio（ローカル）
  └── ファイル操作・検索 → ツール直呼び出し
```

Orchestratorは「何をすべきか」を決めるだけ。重い作業はローカルのsubagentに投げる。Opus本体のトークン消費は、ほぼ計画分だけまで落ちる。

---

## セットアップ手順：3ステップで完結する

### ステップ1：LM StudioでQwen3を起動

LM Studioを起動してQwen3-30B-A3B（MoE、VRAM 24GBで動く）をロードする。ServerタブでローカルAPIを有効にすると、`http://localhost:1234/v1`でOpenAI互換エンドポイントが立つ。

Qwen3を選んだ理由は2つ。32Kコンテキストで長いコードやログを一度に処理できる。thinking mode（/think）で複雑な推論にもそれなりに耐える。

### ステップ2：Claude Codeにカスタムエンドポイントを登録

`~/.claude/settings.json`にLM Studioのエンドポイントを追加する。

```json
{
  "customApiEndpoints": {
    "local-llm": {
      "baseURL": "http://localhost:1234/v1",
      "apiKey": "lm-studio"
    }
  }
}
```

### ステップ3：subagentファイルを作成

`~/.claude/agents/local-worker.md`を作る。

```markdown
---
name: local-worker
description: コード生成・要約・ログ分析など計算負荷の高い下処理
model: qwen3-30b-a3b
apiEndpoint: local-llm
---

あなたはローカルで動く高速ワーカーです。
Orchestratorから与えられたタスクを正確に実行し、
構造化された出力を返してください。
```

あとはClaude Codeが自動でルーティングする。これだけ。

---

## どのタスクをローカルに落とすか：ルーティングの線引き

最初はOrchestrationまで全部ローカルに任せようとした。余裕だろ、と思った。俺の読みが甘かった。計画が破綻した。

Qwen3は優秀だが、長期の依存関係を追いながら複数エージェントを調整する「メタ判断」はOpusに劣る。ここは割り切った。

**ローカル（Qwen3）に落とすタスク：**
- コード生成・リファクタリング（仕様が明確なもの）
- テキスト要約・翻訳
- ログ・データの整形と分析
- 記事下書き・構造化ドキュメント生成
- 繰り返し実行するバッチ処理

**Opus（クラウド）に残すタスク：**
- 複数エージェントのOrchestration
- 曖昧な仕様の解釈
- 最終的なコードレビューと品質チェック
- セキュリティ上のリスク判断

判断基準は1つ。**「この出力が間違っていたとき、後で検知・修正できるか」**。できるならローカル。できないならOpus。

brainに実装しているmodel-routeスキルでは、タスクのメタデータ（complexity・reversibility・output_type）を見て自動選択させている。まだ未検証の部分もあるが、おおよそこの基準で回る。

![]( /images/20260420-2053-claude-code-subagentllm-opus/fig3_routing_decision_tree.png)

---

## 実測：Opus消費が1/30になった夜

brainのautoresearchパイプラインで、実際に走らせた数字を並べる。

**Before（全部Opus）：**
- 1記事分析サイクル：12,400 input tokens / 3,100 output tokens
- コスト：$0.18/サイクル
- 夜間20サイクル実行：$3.60/夜

**After（ローカルルーティング導入後）：**
- Opus消費：410 input tokens / 230 output tokens（Orchestrationのみ）
- Qwen3消費：ローカルなのでAPI課金ゼロ
- コスト：$0.006/サイクル
- 夜間20サイクル実行：$0.12/夜

削減率96.7%。トークン比で約30分の1。

「品質を落とさず削減」と書きたいが、正直に言うと要約精度は微妙に落ちた場面があった。Qwen3はOpusより指示の解釈が硬い。プロンプトを丁寧に書き直す必要があった。

ただし、最終的な記事構造の分析や差別化ポイントの抽出はOpusが見直す。だからアウトプット品質の劣化は感じていない。

レイテンシにも触れておく。Qwen3-30B-A3BはRTX 4090で動かしていて、1,000トークン生成に約8秒。Opusより遅い瞬間もある。夜間バッチは気にならないが、インタラクティブ作業には向かない。

---

## 「ローカルLLMは品質が落ちる」はもう古い前提だ

通説を1つ壊しておきたい。

「ローカルLLMは品質が落ちるから本番では使えない」——1〜2年前ならその通りだった。2026年4月の時点では崩れている。

Qwen3-30Bはコーディングベンチマークで旧世代のGPT-4と遜色ない水準まで来ている。MoE構造で推論コストも低い。これがVRAM 24GBで動くという事実そのものが、去年までの前提を壊している。

ただし、全タスクをローカルに置き換える話ではない。Orchestrationと最終判断はOpusに残す。ここを間違えると品質が崩れる。この使い分けが設計の核心だ。

---

## なぜここまでやるか（本音）

ここだけ本音を書かせてほしい。

僕には脳過敏症がある。課金カウンターが回る感覚に、人より強く反応する。毎回APIを叩くたびに金額が積み上がる感覚、月末の請求への漠然とした不安。これが蓄積すると、AIを使う手そのものが止まる。本末転倒だ。

brainを作ったのは、AIをストレスなく使い倒せる状態を作るためだった。Opusを使うは目的じゃない。良いアウトプットを出し続けることが目的だ。重い下処理はローカルに任せ、Opusは本当に必要な判断だけに使う。

収集。分析。保存。全部ローカルに回せ。

![]( /images/20260420-2053-claude-code-subagentllm-opus/fig4_cost_timeline.png)

---

## まとめ：明日から何をすればいいか

行動レベルまで落とす。

1. **LM StudioをインストールしてQwen3-30B-A3Bをロード**（VRAM 24GBで動く）
2. **`~/.claude/settings.json`にカスタムエンドポイントを1行追加**
3. **`~/.claude/agents/local-worker.md`を作ってsubagentを宣言**
4. **ローカルに落とすのは「出力を後で検証できるタスク」だけに限定**
5. **Orchestrationと最終判断はOpusに残す**

この順で入れれば、今夜からOpus消費は確実に落ちる。

同じ構成で試した人、あるいは試したけどうまくいかなかった人、どんな組み方をしているか教えてほしい。特にQwen3以外のモデル選択や、VRAMが少ない環境での工夫は知りたい。

---

参考: https://reddit.com/r/LocalLLaMA/comments/1sqdg0u/using_qwen36_via_lm_studio_as_a_claude_code/

<!-- figure_style: clean -->

#Claude #ClaudeCode #生成AI #AIでやってみた #ローカルLLM #Qwen3 #LMStudio #subagent #コスト削減 #自律AI
