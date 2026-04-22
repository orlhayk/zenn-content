---
title: "SKILL.mdが暴発する理由｜skill-creatorで12個量産して分かった"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "skill", "ai", "automation", "prompt-engineering"]
published: true
---

skill-creator は便利ツールじゃない。これは「脳の異分身製造装置」だ。

僕は脳過敏症で、細かい判断を連続できない。だから Claude Code 上に brain という自律AIシステムを組んで、夜間に勝手に情報収集・記事生成・配信まで走らせている。その中核にあるのが skill だ。12個量産して、eval を30回以上回して、ようやく気づいたことがある。

**skill が発動しない原因の約8割は、SKILL.md の description 欄の書き方に集約されている。**

公式 skill-creator は文法は教えてくれる。でも「どう設計すれば暴発しないか」は教えてくれない。この記事はそこを埋めるために書く。

## このノートで分かること

- SKILL.md の description を「140字以内・動詞始まり・否定条件つき」で書くと trigger 精度が体感3倍になる理由
- eval が落ちる失敗パターン3類型と、回帰テストの組み方
- skill を hook / MCP / 自律ループに接続して「単体で終わらせない」連携設計
- description 肥大化・trigger 誤発火・eval 偽陽性の具体的な直し方

正直に言うと、最初の5個は全部暴発した。chat 中に勝手に起動して会話の流れを破壊した。その失敗ログを全部さらす。

## なぜ今 skill-creator か — 「脳の異分身」という再定義

Anthropic 公式 skill-creator は、SKILL.md と補助スクリプトを組み合わせて「特定タスク専用の作業手順」を Claude に持たせる仕組みだ。公式ドキュメント([Agent Skills](https://docs.claude.com/en/docs/claude-code/skills))では「再利用可能な知識パッケージ」と説明されている。

でも僕の使い方は少し違う。

skill は「自分の思考パターンの外部化装置」だ。脳過敏症で毎回同じ判断を再演算するのがキツい。だから「記事ネタ選定の判断基準」「販売物テンプレの生成手順」「配信前チェックリスト」を skill に落とし込んで、Claude に代わりに持たせる。

考えなくていい領域を増やす。記録する。再利用する。

この3つが揃うと、1日の判断エネルギーが劇的に減る。

## SKILL.md 設計の原則 — description が9割

SKILL.md の frontmatter はこうなる。

```yaml
---
name: note-topic-check
description: note記事のネタ重複を過去14日分でチェック。同一軸が週3本以上なら停止を提案する。コード生成タスクでは発動しない。
---
```

ここで重要なのは description。僕が12個作った結果、効く description には共通パターンがあった。

**ルール1: 動詞始まり。**「〜をする」で始める。「〜のためのツール」みたいな名詞句は trigger が曖昧になる。

**ルール2: 140字以内。** 長くすると Claude 側の判定が甘くなり、無関係な場面で発動する。試しに300字書いた skill は、普通の雑談中に4回暴発した。

**ルール3: 否定条件を入れる。** 「〜では発動しない」を明記する。これだけで誤発火率が体感で3分の1になる。

## eval 最適化の実務 — 落ちる原因は3類型しかない

skill-creator には eval（自動テスト）機能がある。`skill eval` で回すと、想定シナリオで skill が正しく発動するかを検証できる。

僕が30回以上回して分かった失敗類型はこれだ。

### 類型1: description が抽象的すぎて発動しない（偽陰性）
「効率的に処理する」みたいな文言は Claude が trigger を掴めない。具体名詞と具体動詞に書き換える。

### 類型2: 類似 skill と競合して別の skill が呼ばれる（誤発火）
name と description を並べて grep レベルで衝突チェックする。僕は `note-write` と `note-topic-check` が競合して、記事生成したいときに毎回ネタチェックが走った。

### 類型3: 引数の型が曖昧で実行時にコケる（真陽性だが失敗）
SKILL.md 内の手順記述で、入力想定を「URL または記事ID」みたいに OR で書くと、Claude が混乱する。「URL のみ受け付ける、ID の場合は拒否」と明示する。

回帰テストは、成功ケース3個・失敗期待ケース3個を最小セットとして固定。skill 更新のたびに両方走らせる。これやらないと、1つ直して3つ壊れる。

## チーム運用とバージョニング — 命名規則で事故を防ぐ

個人運用でも skill が5個超えた時点で名前が衝突し始める。僕はこの命名規則で整理した。

- `{domain}-{action}` 形式固定（例: `note-write`, `research-collect`）
- domain は最大2階層（`note` / `note.pipeline` まで）
- 依存 skill は SKILL.md の末尾に `# Depends` セクションで明記

バージョニングは git のタグで管理。skill ディレクトリごと `skills/note-write/` に切って、変更は PR ベースで回す。小さい単位で配布するほうが、後でロールバックが効く。

## hook / MCP / 自律ループとの接続

ここが公式ドキュメントに書かれていない領域。

僕の brain では、skill 単体では動かさない。必ず hook と MCP と組み合わせる。

- **hook 連携**: PostToolUse hook で skill 実行後の副作用（ログ記録・通知）を自動化。skill 側に副作用を書かない。
- **MCP 併用**: 外部データ取得は MCP server に分離。skill は「判断と生成」に専念させる。
- **自律ループ組み込み**: cron で起動する brain のメインループから、skill を条件分岐で呼び出す。skill は「呼ばれる側」に徹する。

![]( /images/20260423-0401-claude-code-skill-creator-12/fig3_skill_hook_mcp_architecture.png)

この分離をやると、skill の責務が極端に小さくなる。小さいほど eval が安定する。安定するほど量産できる。

## トラブルシューティング集

**description 肥大化**: 200字超えたら即リファクタ。補助情報は SKILL.md 本文に移す。

**trigger 誤発火**: 否定条件を追加。それでもダメなら name 自体を変える。`general-helper` みたいな汎用名は即死ぬ。

**eval 偽陽性**: 成功判定の条件が緩すぎる。出力の特定文字列を必須にする。

## まとめ: skill は「書く」より「育てる」

skill-creator は一度作って終わりじゃない。eval ログを溜めて、暴発を観察して、description を縮めて、否定条件を足す。

書いた。回した。直した。

この3つを週次で回すと、skill は生き物みたいに精度が上がる。僕の note-pipeline も、最初の版から6回書き直してようやく安定した。

ただし、全てを skill 化するのは罠だ。単発で済むタスクは普通に Claude に頼めばいい。skill は「3回以上繰り返す判断」に限定する。この線引きを守らないと、skill 地獄に落ちる。

同じように skill を量産してる人いたら、暴発パターンと直し方を教えてほしい。俺はまだ12個で、brain の自律化はこれからが本番だ。

試す入り口は整っている。[Anthropic 公式 skill docs](https://docs.claude.com/en/docs/claude-code/skills) と [skill-creator リポジトリ](https://github.com/anthropics/skills) を横に置いて、まずは1個作って eval を5回回してみてほしい。description を2回書き直した頃に、この記事の話が腹落ちするはず。

---
参考: https://qiita.com/TaichiEndoh/items/8b8ed06bb76a80bb34c2

#Claude #ClaudeCode #生成AI #AIでやってみた #skillcreator #SKILLmd #エージェント設計 #自律AI #MCP #Anthropic

<!-- figure_style: clean -->
