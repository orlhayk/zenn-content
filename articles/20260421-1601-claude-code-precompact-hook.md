---
title: "Claude Code の PreCompact hook で『幻覚編集』を封じた話"
emoji: "🐛"
type: "tech"
topics: ["claude", "claudecode", "ai", "debugging", "automation"]
published: true
---

3時間かけて積み上げたリファクタが、たった一言で吹き飛んだ。
「`src/utils/retry.ts` を編集しました」——そのファイルは、存在しない。
`ls` しても、`git status` しても、どこにもない。でも Claude Code は確信を持って「編集した」と言う。

正直に言うと、最初は自分が寝ぼけてるのかと思った。でも違った。これは context compaction が走った直後に起きる、構造的な幻覚だ。

今日はその幻覚を、v2.1.83 で追加された **PreCompact hook** と CLAUDE.md への防御指示、そして git チェックポイントの3段構えで封じた話をする。brain（自律AIシステム）を夜間自走させてる身として、これは生産性の問題じゃなく、生存の問題だった。

## このノートで分かること

- なぜ context compaction が「存在しないファイルの編集」を引き起こすのか
- PreCompact hook で圧縮直前に git チェックポイントを打つ実装
- CLAUDE.md に「自分の記憶を信用するな」と書く防御パターン
- hook + git + memory をパイプライン化した運用構成
- 長時間セッションを安全に回すための3つのルール

## Claude Code が突然、存在しないファイルを「編集した」と言い出す

現象はこうだ。2時間以上のセッションで context が圧縮されたあと、次のターンで Claude が `Edit` ツールを呼ぶ。返答では「`src/auth/session.ts` の L42 を修正しました」と言う。でも実際のツールコールログを追うと、触ったのは `src/auth/token.ts`。ファイル名が混入・捏造されている。

これは確認不足じゃない。圧縮アルゴリズムが会話を要約するとき、複数のファイルパスが1つのトークン列に畳み込まれる過程で発生する。GitHub Issue #46602 でも報告されている既知の挙動だ（※Issue番号は元記事準拠、未検証）。

僕は brain で夜間12時間の自走ループを回している。朝起きて差分を見たら、**存在しないパスへの編集履歴が3件、commit log に残っていた**。中身は正しい。ファイル名だけが嘘。これほど質の悪い幻覚はない。

## なぜ起きる？context compaction と幻覚の仕組み

Claude Code の context は 200K トークン。それを超えそうになると、過去のやりとりを要約して圧縮する。このとき起きるのが「**ファイル名の同化**」だ。

通説ではこう言われる。「AI の幻覚は確率的な揺らぎ、プロンプトを丁寧に書けば減る」——嘘だ。
少なくとも context compaction 後の幻覚は、プロンプト品質では解決しない。圧縮が走る瞬間、Claude は**自分の過去の発言を参照点として信じ込む**。そこに混入したノイズは、次のターンで「事実」として扱われる。

ただし、これは Claude Code 固有の問題ではない。長い context を扱う LLM 全般で起きる。だから「圧縮後は自分の記憶より、外部の事実を信用させる」という設計思想が必要になる。

## PreCompact hook で圧縮直前に git チェックポイントを自動作成する

PreCompact hook は、compaction が走る**直前**にシェルコマンドを挟める仕組みだ。ここに git commit を差し込む。

`~/.claude/settings.json` の hooks 設定（抜粋）:

```json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "cd \"$CLAUDE_PROJECT_DIR\" && git add -A && git commit -m \"checkpoint: pre-compact $(date +%Y%m%d-%H%M%S)\" --allow-empty || true"
          }
        ]
      }
    ]
  }
}
</code>

これで圧縮前の状態が必ず git に残る。幻覚が起きても `git log` を見れば「実際に何が変わったか」が検証できる。brain では 1セッションあたり平均 4.2 回の圧縮が走るので、チェックポイントが自動で4つ積まれる計算。

## CLAUDE.md に「自分の記憶を信用するな」と命令する

hook だけだと防御は半分。圧縮後の Claude 自身にも、防御姿勢を取らせる必要がある。

僕の CLAUDE.md にはこう書いてある:

> context compaction 後の最初のターンでは、記憶している直近のファイル操作を **必ず git log と git diff で検証せよ**。自分の発言内にあるファイルパスを参照するな。検証なしにファイル名を出力することを禁止する。

これを書く前は、圧縮後の幻覚率が体感で週3〜4回。書いた後は、ほぼゼロ。正確に測ったわけじゃない（ここは未検証）けど、体感は劇的に変わった。

短い命令を、断定で。これが効く。Claude に「〜してください」じゃなく「禁止する」と書く。ここは俺の経験則だ。丁寧に書くほど無視される。強く書くほど守る。

## hook + git + memory のフルセット構成

僕が brain で使ってる幻覚防御パイプラインは3層だ。

1. **PreCompact hook**: 圧縮前に git checkpoint を自動作成
2. **CLAUDE.md 防御指示**: 圧縮後の最初のターンで git 検証を強制
3. **PostToolUse hook**: Edit/Write ツール実行後にファイル存在検証スクリプトを走らせる

3つめの PostToolUse はこんな感じ:

```json
{
  "PostToolUse": [
    {
      "matcher": "Edit|Write",
      "hooks": [
        { "type": "command", "command": "test -f \"$CLAUDE_TOOL_FILE_PATH\" || echo '⚠️ ファイル不在検出' >&2" }
      ]
    }
  ]
}
```

存在しないファイルへの Edit を Claude が宣言した瞬間、stderr に警告が出る。これで幻覚の現行犯を押さえられる。

収集。検証。記録。この3ステップを自動化したことで、brain の夜間自走は「朝起きて成果を確認する」運用に変わった。差分を信用する。発言を信用しない。これが原則。

![]( /images/20260421-1601-claude-code-precompact-hook/fig3_three_layer_defense.png)

## 長時間セッションを安全に回すための3つの運用ルール

最後に、hook を入れたうえで僕が守ってる運用ルールを3つ。

**① 2時間ごとに明示的 `/compact` を自分で打つ**
自動圧縮のタイミングは読めない。だから 2時間ごとに手動で走らせて、圧縮後の状態を自分の目で確認する。brain では cron で通知を飛ばしてる。

**② checkpoint commit には必ずタイムスタンプを入れる**
「checkpoint: pre-compact」だけだと log が埋もれる。日付時刻を必ず含めれば、あとで `git log --grep` で特定できる。

**③ 幻覚を見つけたら session を閉じて再起動**
一度幻覚が出たセッションは、もう信用しない。`/clear` じゃなく完全に新規セッションを立ち上げる。汚染された context を引きずるより、再起動のコストのほうが安い。

---

## おわりに

AI の幻覚は、自信満々なほど質が悪い。Claude Code は優秀だからこそ、context compaction の瞬間に静かに嘘をつく。でも hook と git を組み合わせれば、その嘘を**事前に無効化**できる。

僕は脳過敏症で、自分の記憶も当てにならない日がある。だから AI の記憶も当てにしない。差分を信じる。log を信じる。ファイルシステムを信じる。

同じように Claude Code を長時間走らせてる人で、「存在しないファイルを編集された」経験ある人、hook どう組んでるか教えてほしい。brain の構成はまだ磨きたいところが山ほどある。

---
参考: https://qiita.com/yurukusa/items/a4a79ee057de1e532ff3

<!-- figure_style: clean -->

#Claude #ClaudeCode #生成AI #AIでやってみた #PreCompactHook #contextcompaction #自律AI #hook設計 #CLAUDEmd #セッション管理
