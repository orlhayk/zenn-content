---
title: "CLAUDE.md は「指示書」ではなく「憲法」だった"
emoji: "📜"
type: "tech"
topics: ["claude", "agent", "automation", "productivity", "prompt"]
published: true
---

CLAUDE.md を書き直した。

きっかけは Claude Code 公式の [Agent Teams ドキュメント](https://code.claude.com/docs/en/agent-teams) を読んだ瞬間だった。そこにはこう書いてある。**CLAUDE.md は repository の「constitution（憲法）」である**。

この一文で、自分の CLAUDE.md の扱い方が根本から間違っていたことに気づいた。

僕は40個のskill、12個のhook、5個のMCPを抱える自律AIシステム「brain」を運営している。CLAUDE.md は今まで「AIへの指示書」のつもりで書いていた。**だから長くなるし、毎日ブレる。就業規則より詳細な実務マニュアルを書こうとしていたから**だ。

**CLAUDE.md は指示書じゃなく、憲法。就業規則は skill に書く。実務マニュアルは hook と MCP に落とす。**

この3層が理解できた瞬間、brain の統制が変わった。

## このノートで分かること

- Claude Code 公式が「CLAUDE.md = 憲法」と定義した意味
- CLAUDE.md / skill / hook / MCP の4層統制モデル
- 40 skill を抱える brain を再編成したときに削れた300行
- Agent Teams 機能で複数AIが協働する時代の CLAUDE.md 設計
- 個人開発者が今日からできる CLAUDE.md 3層分離テンプレ

## 公式が示した「CLAUDE.md = 憲法」

一次ソースは [Claude Code 公式 Agent Teams ドキュメント](https://code.claude.com/docs/en/agent-teams) と、これを解説した [The CLAUDE.md Configuration Hierarchy](https://agentfactory.panaversity.org/docs/General-Agents-Foundations/claude-code-teams-cicd/claude-md-configuration-hierarchy) だ。

公式の定義を要約するとこうなる。

- CLAUDE.md = repository 全体の context、アーキテクチャ、標準、ビルドコマンド
- Agent が teammate を spawn するとき、同じ CLAUDE.md をロードする
- **3人の teammate が CLAUDE.md を読むコストは、3人がコードベースを個別探索するコストより遥かに安い**

ここに示されているのは、「CLAUDE.md は人間のためのドキュメント」ではなく、「**組織内の全エージェントが同じ前提を共有するための契約書**」であるという思想だ。

憲法と呼ぶのは誇張じゃない。憲法が機能する組織では、部門ごとの施行規則（= skill）や日次オペレーション（= hook / MCP）が憲法と整合する。憲法がない組織では、部門が勝手に動き始めて、統制が効かなくなる。

## 4層統制モデル

brain を再編成するとき、Claude Code の機能を以下の4層にマッピングした。

**第1層: CLAUDE.md（憲法）**
- プロジェクトの目的
- アーキテクチャの大枠
- 禁止事項（破ると組織が壊れるルール）
- 命名規則・ディレクトリ構造
- **頻度低・変更コスト高・粒度粗い**

**第2層: skill（就業規則）**
- 部門別・職種別の手順書
- 「この作業ならこの流れで」のパターン集
- **頻度中・変更コスト中・粒度中**

**第3層: hook（実務マニュアル）**
- イベント駆動で自動実行される処理
- PreToolUse / PostToolUse / Stop で挟む「組織の自動反応」
- **頻度高・変更コスト低・粒度細かい**

**第4層: MCP（外部部門）**
- 外部サービスとの接続（DB / ブラウザ / ファイルシステム / API）
- 組織の「庶務」や「IT部門」に相当
- **頻度高・変更コスト高・粒度極細**

この4層に分けると、何をどこに書くべきか迷わなくなる。迷わなくなると、CLAUDE.md は自然と短くなる。

## brain の CLAUDE.md が300行削れた

僕の brain の CLAUDE.md は元々750行あった。書き直したら420行になった。**330行が skill / hook / MCP に移動した**。

具体例を挙げる。

削ったもの（skill に移した）:
- note 記事の書き方テンプレ → `note-write.skill`
- 図の生成基準（フォントサイズ・色パレット） → `note-thumb.skill`
- ベンチ系記事のソース調査手順 → `note-source-research.skill`

削ったもの（hook に移した）:
- セッション終了時の学習候補抽出 → `Stop hook + learning-rules`
- テーブル編集時の validation → `PreToolUse hook`

削ったもの（MCP に移した）:
- note.com の下書き投稿操作 → 既存の `note_publisher.py` に集約

CLAUDE.md に残ったもの:
- brain の目的（ノウハウ販売チャネル構築）
- 30日ロック期間の宣言
- モデルルーティングの原則（Haiku 優先 / skill 先行）
- 禁止事項（他IP派生・ロック中の軸変更）

**憲法的な粒度のものだけが残った**。読み直してみると、実際に組織憲法のような手触りがある。

## Agent Teams 時代の CLAUDE.md

ここから先が本題だ。

Claude Code の Agent Teams 機能は、1つの lead agent が複数の teammate を spawn し、並列で同じ codebase を触る。**teammates 全員が同じ CLAUDE.md を読む**。

これは何を意味するか。

CLAUDE.md の曖昧な一文は、teammate の数だけ解釈の揺らぎを生む。3人の teammate が同時に同じ CLAUDE.md を読んで、3通りの解釈で動いたら、組織は分裂する。

だから Agent Teams 時代の CLAUDE.md は、**個別の実装判断を減らすほど強くなる**。「こういうときはこう」は skill に移す。憲法には「目的」と「禁止」だけ残す。

公式ドキュメントが何度も強調する `mailbox system`（agent 間の直接メッセージング）も同じ思想だ。teammate 同士が勝手に communicate するなら、憲法は共通基盤として機能していなければならない。

## 今日からできる CLAUDE.md 3層分離テンプレ

個人開発者が手を動かせる形で置く。

**Step 1**: 今の CLAUDE.md を開いて、各段落を以下の4色で色分けする。

- 🔴 憲法 (目的・禁止・大枠アーキテクチャ) → CLAUDE.md 残留
- 🟡 就業規則 (作業手順・パターン) → skill へ
- 🟢 実務マニュアル (自動化可能な反応) → hook へ
- 🔵 外部連携 (API/DB/Browser) → MCP へ

**Step 2**: 🟡🟢🔵 を対応する skill / hook / MCP ファイルに移動。

**Step 3**: CLAUDE.md に残った 🔴 を読み返して、「これは組織の憲法として読めるか」を自問。読めなければさらに削る。

この手順で brain の CLAUDE.md は750行→420行になった。削れただけでなく、**新しい skill を書くときに「これは skill か hook か」を迷わなくなった**。

## 公開リソース

この再編成で参考にしたもの:

- [Claude Code 公式: Agent Teams](https://code.claude.com/docs/en/agent-teams) ― teammate/mailbox の公式仕様
- [FlorianBruniaux/claude-code-ultimate-guide](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) ― Agent Teams のワークフロー解説
- [CLAUDE.md Configuration Hierarchy](https://agentfactory.panaversity.org/docs/General-Agents-Foundations/claude-code-teams-cicd/claude-md-configuration-hierarchy) ― 階層モデルの詳細
- [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) ― Claude Code 設計思想の体系解剖

## まとめ: 指示書を書くのをやめた日

整理する。

- CLAUDE.md は Claude Code 公式が「repository の憲法」と定義している
- 4層統制モデル: CLAUDE.md（憲法）/ skill（就業規則）/ hook（実務）/ MCP（外部）
- brain の CLAUDE.md は750行→420行に削れた。330行は他の層に移動
- Agent Teams 時代は teammate 全員が同じ CLAUDE.md を読むので、憲法的粒度が必須
- 今日からできる3層分離: 各段落を🔴🟡🟢🔵で色分けして移動

「CLAUDE.md が長くなって困ってる」と相談されたら、最近は「それ指示書になってませんか」と返すようにしている。指示書を書くのは楽しい。だが憲法を書く方が、遥かに強い組織ができる。

brain の CLAUDE.md 最新版や、skill の分類基準をまとめた一覧は、まだ自分の手元で検証中だ。整理できたら記事として残したい。

同じように CLAUDE.md を育てている人、4層分離の実例を教えてほしい。僕は hook と MCP の境界でまだ迷ってる。

---

参考:
- [Claude Code 公式: Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [The CLAUDE.md Configuration Hierarchy](https://agentfactory.panaversity.org/docs/General-Agents-Foundations/claude-code-teams-cicd/claude-md-configuration-hierarchy)

<!-- figure_style: clean -->

#Claude #ClaudeCode #CLAUDEmd #AIエージェント #AgentTeams #自律AI #Anthropic #生成AI #AI開発 #マルチエージェント
