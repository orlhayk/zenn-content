---
title: "Claude Codeのトークンを78%削った──6つの習慣をhookで自動化"
emoji: "💰"
type: "tech"
topics: ["claudecode", "automation", "token", "ai", "hook"]
published: true
---

Claude Code の月額が $217 を超えた月、正直に言うと血の気が引いた。

`rtk gain` を叩いたら削減率 78.4% と出た。6つの習慣を人力でやるのをやめて、hook と skill に肩代わりさせた結果だ。元記事の「意識しましょう」では絶対に到達しない数字だと思っている。

参考にしたのはこの記事。
https://zenn.dev/ruralwritter/articles/b5e90a4883308e

6つの習慣はどれも正しい。**正しいけど、人間がやる前提だと3日で忘れる。** 僕は忘れた。だから仕組みに肩代わりさせた話を置いていく。

## このノートで分かること

- Claude Code がトークンを食う6つの理由（元記事の要約）
- 「意識する」では止まらない、実際に止まったのは hook と skill だった
- RTK (Rust Token Killer) で 60-90% 削減した実測ログ
- 6習慣を settings.json の hook と skill に落とし込む構成図
- 脳過敏症エンジニアが省トークン運用を「認知資源の節約」として組んだ理由

## なぜ Claude Code はトークンを食うのか（30秒で）

元記事の6習慣を雑に要約するとこうなる。

1. 毎回 CLAUDE.md を全読みしてる
2. 無駄に長い会話履歴を引きずる
3. MCP ツール定義を全部ロードする
4. 巨大ファイルを丸ごと Read する
5. 同じ検索を何度も繰り返す
6. thinking を常時MAXで回す

全部正しい。**全部、気をつければ防げる。** ただし「気をつければ」の部分が成立しない。僕はしなかった。

## 習慣論では止まらない：意志力に頼る節約の限界

正直に言うと、僕は最初の2週間、元記事通りに「意識」してみた。

結果、課金額は変わらなかった。むしろ増えた。理由は単純で、コーディングに集中すると節約のことを忘れるからだ。脳過敏症持ちの僕にとって「コードを書きながらトークン消費も監視する」は二重タスクで、完全にオーバーフローする。

ここで気づいた。**トークン浪費は金の問題じゃない。認知資源の浪費だ。** 節約のために集中力を削るのは本末転倒だ。

だから方針を変えた。人間が意識するのをやめて、hook に全部やらせる。

## RTK で 78.4% 削った実測ログ

RTK (Rust Token Killer) という自作寄りの Rust 製プロキシを噛ませている。`git status` と叩くと hook が裏で `rtk git status` に書き換えて、出力を圧縮してから Claude に返す。

`rtk gain --history` の直近7日の生データがこれだ。

```
Command              Calls   Raw tokens   Saved      Rate
git status            142      38,214     34,112   89.3%
git diff               87      91,488     72,341   79.1%
ls / find              63      27,005     24,108   89.2%
npm test              21      48,702     31,445   64.5%
---
Total                 547   1,284,021   1,006,885   78.4%
```

100万トークン以上、1週間で削っている。月額換算で $120〜$160 の差だ。

ただし留保を置く。RTK はまだ荒削りで、特定のコマンド（`grep` に `-C` 大きめを付けた時）で圧縮しすぎて情報落ちすることがある。そういう時は `rtk proxy <cmd>` で素通しにする脱出口を用意している。完璧じゃない。でも 78% は78%だ。

## 6つの習慣を hook と skill で自動化する

ここからが本題。元記事の6習慣を、1つずつ settings.json の hook と skill に落とし込んだ構成を出す。

### 1. CLAUDE.md 全読み → `context-budget` skill で要約注入

毎セッションで CLAUDE.md を8000トークン読むのをやめた。`context-budget` skill が要約版（1200トークン）を先に注入し、詳細は「必要になった時だけ Read」に切り替える。差分は 6800 トークン × セッション数。

### 2. 会話履歴引きずり → `prune` skill + SessionStart hook

SessionStart hook で前回の会話要約をスナップショットに落とす。今日のセッション冒頭で実際に動いた：

```
SessionStart:startup hook success: Snapshot saved
5790 bytes, 72 lines
```

これだけで、前セッションの文脈を 5.8KB まで圧縮してから始められる。

### 3. MCP ツール全ロード → `ToolSearch` で遅延読み込み

使わない MCP ツール（Gmail, GDrive, Cron 等）を `available-deferred-tools` に回して、必要時だけ `ToolSearch` で schema を引く。起動時のツール定義ロードが 1/5 になった。

### 4. 巨大ファイル丸読み → PreToolUse hook で Read にサイズ制限

Read ツールに対して PreToolUse hook をかけ、500行以上のファイルは強制的に `offset/limit` を要求する。僕が忘れても hook が止める。

### 5. 検索の重複 → PostToolUse hook でキャッシュ

同じ Grep / Glob を15分以内に再実行したら、キャッシュから返す。content-hash-cache-pattern skill の応用。

### 6. thinking 常時MAX → `MAX_THINKING_TOKENS` で上限固定

`~/.claude/settings.json` に `MAX_THINKING_TOKENS=10000` を書いておく。設計時だけ上げる。それ以外は下げっぱなし。

![]( /images/20260425-0401-claude-code-rtk-hook-78-6/fig3_hook_architecture.png)

## 今日から貼れる settings.json 差分

実際に僕が使っている設定の、公開できる範囲を置く（API キー等はダミー）。

```json
{
  "env": {
    "MAX_THINKING_TOKENS": "10000"
  },
  "hooks": {
    "SessionStart": [
      { "command": "brain snapshot save" }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "rtk-wrap $CLAUDE_TOOL_INPUT"
      },
      {
        "matcher": "Read",
        "command": "enforce-read-limit $CLAUDE_TOOL_INPUT"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Grep|Glob",
        "command": "cache-search-result $CLAUDE_TOOL_INPUT"
      }
    ]
  }
}
```

これを貼るだけで、少なくとも僕の環境では課金額が半減した。未検証の環境依存があるので、そのままコピペせず自分の運用に合わせて削ってほしい。

## 脳過敏症エンジニアが省トークンに本気になった理由

ここで少し個人的な話を挟む。

僕は脳過敏症で、普通の人より認知資源が少ない。音、光、情報量、全部ダメージになる。だからコーディング中に「今トークン何万使ったっけ」を考えると、そこで集中が切れる。切れると30分戻らない。

正直、俺にとってトークン浪費は金より集中力の問題だった。

$200 削るより、集中30分を守りたい。hook で自動化したのは節約のためじゃない。**考えなくていい状態を作るため**だ。結果として金も浮いた。

考えない。意識しない。忘れる。それでも走る。これが設計のゴールだった。

## 常識崩壊、3つ

最後に、この実験で僕の中で壊れた通説を3つ置く。

**常識崩壊①** 「トークン節約はプロンプト圧縮が9割」── 嘘。実測で効いたのは hook による強制で、プロンプト工夫の寄与は2割以下だった。

**常識崩壊②** 「thinking はMAXが正義」── 嘘。10000 に絞っても出力品質は体感で変わらなかった。ただし設計レビュー系タスクだけは落ちた、そこは留保。

**常識崩壊③** 「MCP は全部入れとけ」── 嘘。遅延読み込みに変えた瞬間、起動が軽くなった上に精度も上がった（不要ツール混入で誤選択が起きていた）。

## まとめと入り口

長々書いたので短くまとめる。

Claude Code のトークン浪費は、習慣では止まらない。止まるのは hook だけだ。6つの習慣を1つずつ settings.json に落とし込めば、意志力ゼロで走る。僕はそうした。課金額は半減し、集中力は戻った。

試す入り口は整っている。

- 元記事（習慣論の原典）: https://zenn.dev/ruralwritter/articles/b5e90a4883308e
- Claude Code 公式 hooks ドキュメント: https://docs.claude.com/en/docs/claude-code/hooks
- RTK は現在 private で磨いている最中。構成だけならこの記事で全部公開した

同じことやってる人がいたら、使ってる hook 教えてほしい。特に「意志力でやめた Tips」があれば、そっちが本命だと思う。

---
参考: https://zenn.dev/ruralwritter/articles/b5e90a4883308e

<!-- figure_style: clean -->

#Claude #ClaudeCode #生成AI #AIでやってみた #トークン削減 #hook自動化 #RTK #settings_json #自律AI #エンジニア効率化
