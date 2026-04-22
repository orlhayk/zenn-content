---
title: "3エージェント Harness で 6時間$200の自律コーディング"
emoji: "🧠"
type: "tech"
topics: ["claude", "agent", "anthropic", "llm", "ai"]
published: true
---

単独で走らせていた brain が、急に遅く見え始めた。

きっかけは Anthropic が2026年4月に公開した [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) だ。**Planner / Generator / Evaluator の3エージェント構成で、6時間走りきって$200、単独走行（20分・$9）では届かない完成度の製品を出した**という。

倍率で言うと時間18倍、コスト22倍。だが成果物の到達点が別物だった。

昨日の記事では「skill を書けば Haiku で Opus を超える」と書いた。それは個の最適化の話だった。今日は**組織化**の話をする。

**skill 設計が「兵」の強化なら、Harness 設計は「部隊」の編成だ。**

## このノートで分かること

- Anthropic 3エージェントHarness（Planner/Generator/Evaluator）の核心
- なぜ「context 圧縮」より「context リセット + 構造化 handoff」が勝つのか
- 単独走行 brain を 3-agent 構成に移した時に起きたこと
- Playwright MCP を Evaluator に組ませる具体的なパターン
- 個人開発者が今日試せる最小構成

## 3エージェントHarness の構成

一次ソースは Anthropic 公式エンジニアリングブログ [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) と、[2026 Agentic Coding Trends Report](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf)。実装を追いかけた [InfoQ 記事](https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/) が構造を図解している。

役割分担はこうだ。

**Planner（計画者）**
- プロダクト仕様を「独立して完結する小さなタスク」に分解する
- 各タスクが自己完結していることが重要（後続が context 持ち越しなしで実行できる）

**Generator（実装者）**
- 1タスク = 1 context window で実装する
- 次のタスクに渡す前に「構造化された引き継ぎ資料」を書く
  - JSON の feature spec
  - commit-by-commit の進捗
  - `claude-progress.txt`

**Evaluator（評価者）**
- 事前に定義された基準でアウトプットを採点
- フロントエンドなら Playwright MCP で実際のブラウザ操作を回して UI/UX を検証
- 合格しなければ Generator に差し戻す

## 核心は「context リセット」

個人的に一番刺さったのは、**context compaction ではなく context reset を選んだ**点だ。

これまで長時間セッションの常識は「compaction で要約して圧縮を繰り返す」だった。Anthropic の主張はこうだ。**圧縮は劣化を残す。長時間セッションでは劣化が累積する。context を丸ごとクリアして、構造化された state artifact だけ渡した方が安い。**

トークン単価は上がる。でも劣化由来のやり直しがなくなるので、トータルコストで勝つ。

これは自分でも実感があった。brain の夜間12時間ループで、context 圧縮が3-4回走ったあとの出力は明らかに質が落ちる。ファイル名がブレる、過去の決定を忘れる。あれは圧縮のせいだったのかもしれない、と読みながら手がぞわっとした。

## 単独走行brainを3-agent化してみた

僕の brain は昨日までずっと単独走行だった。1つの Claude Code セッションが note 執筆から X 投稿下書きまで全部やる構成。

今朝、Anthropic の Harness を参考に分解してみた。

**Planner 層（僕が実装）**
- 既存の `topic_score` ジョブを拡張
- 記事ごとに「タスクA: ソース調査 / タスクB: 構成案 / タスクC: 本文執筆 / タスクD: 図生成 / タスクE: 下書き投稿」の5分解
- 各タスクの完了条件を JSON で書き出す

**Generator 層**
- 既存の `note_draft` / `figures_gen` ジョブを単タスク専用に縛る
- タスク間の引き継ぎは JSON + `article-progress.json` ファイル経由
- 1タスク = 1 Claude Code 起動（context 使い切ったら reset）

**Evaluator 層**
- 新規 `note_evaluator` ジョブを追加
- 「タイトルが素人に通じるか / 数字が2つ以上あるか / 公開repoが挙がっているか」のチェックリスト
- 不合格なら Generator に差し戻し

まだ1日目なので定量評価は出せない。ただ体感では、**記事ごとのブレが減った**。Evaluator が「あなたのタイトル、専門用語から始まってる」と弾いたので書き直した。今日のこの記事も、初稿は「3エージェントHarness が〜」で始まっていた。

## Playwright MCP を Evaluator に組ませる

フロントエンド実装を評価するときの Anthropic の構成は、そのまま応用できる。Evaluator が Playwright MCP 経由で:

1. ビルドされたアプリを起動
2. ブラウザを操作（クリック/入力）
3. 期待される挙動が出るか検証
4. 出なければ Generator にスクリーンショット付きで差し戻し

`claude mcp` コマンドで Playwright MCP を差し込めば、この構成は個人でも組める。僕の brain の次の一手はこれだ。note 記事の「投稿後プレビュー」を Evaluator に自動チェックさせる。

## 公開リソースで今日試せる最小構成

3-agent まで行かなくても、**Planner / Evaluator の2レイヤーを足すだけ**で効果は出る。

- [Anthropic 公式: Agent Teams ドキュメント](https://code.claude.com/docs/en/agent-teams) ― Team lead / teammates / mailbox の公式仕様
- [wshobson/agents](https://github.com/wshobson/agents) ― Claude Code 向けマルチエージェント OSS
- [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) ― Claude Code のエージェント構造を体系的に解剖した論文つきリポ

この3つを読めば、Harness 実装の90%は見える。

## なぜ「20分・$9」で満足してはいけないか

ここが今日一番言いたいことだ。

単独走行は速くて安い。だが「**完成度が低いことに気づけない**」という致命的な問題がある。自分で作って自分で評価する agent は、自分の盲点を構造的に認識できない。

Evaluator を外部に置くことで初めて、「この UI は言われた通り動くが、ユーザー体験が悪い」「このコードは動くが、エッジケースで壊れる」が検出される。

人間の組織と同じだ。レビュアーを置かない開発チームは速いが、品質は落ちる。Anthropic が $200 払って6時間回すのは、単なる金持ちの道楽ではなく、**評価を外部化した構造の必然的なコスト**だ。

個人開発で $200 は重い。でも Planner と Evaluator を入れる構造的判断はコピーできる。コピーすれば、成果物の質が変わる。

## まとめ: 個の最適化の先に、組織化がある

整理する。

- Anthropic 3-agent Harness は Planner/Generator/Evaluator で分業、6時間$200で単独走行では届かない完成度を出した
- 勝因は context 圧縮でなく context リセット + 構造化 handoff
- 個人でも Planner と Evaluator の2レイヤーを足すだけで成果物の質が変わる
- Playwright MCP を Evaluator に組ませれば、フロントエンド実装の自動評価も可能
- 昨日の「skill = 兵の強化」の先に、「Harness = 部隊の編成」がある

brain で動かしている3-agent 化の実装（`article-progress.json` のスキーマや Evaluator のチェックリスト）は、まだ1日目で検証途中だ。安定してきたら記事として残したい。

同じように個人でマルチエージェント組んでる人、構成を教えてほしい。Evaluator の粒度設計で迷ってる。

---

参考:
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [Anthropic: 2026 Agentic Coding Trends Report](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf)
- [InfoQ: Anthropic Designs Three-Agent Harness](https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/)
- [Claude Code 公式: Agent Teams](https://code.claude.com/docs/en/agent-teams)

<!-- figure_style: clean -->

#Claude #ClaudeCode #AIエージェント #Harness #自律AI #Anthropic #マルチエージェント #AgentTeams #生成AI #AI開発
