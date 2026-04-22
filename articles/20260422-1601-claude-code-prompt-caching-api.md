---
title: "Claude Code の Prompt Caching で API コスト 1/8 削減"
emoji: "⚡"
type: "tech"
topics: ["claude", "prompt-caching", "cost-optimization", "llm", "performance"]
published: true
---

1 ターンあたり 20,000 トークン。10 ターンで 200,000 トークン。同じ system prompt と tool 定義を、俺のエージェントは毎回まっさらな頭で読み直していた。

Prompt Caching を入れた瞬間、自律 brain ループの API コストは **1/8** になり、初動レイテンシは **4 秒から 0.6 秒** に縮んだ。差を生んだのは新しいモデルでも高性能な GPU でもない。「どこにキャッシュ境界を置くか」というたった一つの設計判断だった。

正直に言うと、最初の3週間、俺はキャッシュが効いていると思い込んでいた。効いていなかった。system prompt の末尾にタイムスタンプを1行入れていただけで、20,000 トークン分のキャッシュは毎ターン吹き飛んでいた。

この記事は、Zenn の良記事「[AIエージェントは毎ターン、同じ20,000トークンを読み直している](https://zenn.dev/analysis/articles/thought-analyzer-prompt-caching)」を読んで、自分の brain 実装に当てはめて検証した記録だ。

## このノートで分かること

- Prompt Caching は最適化テクではなく **設計規律** である理由
- キャッシュを一瞬で破壊する **5 つのアンチパターン**
- Claude Code 実装者のための skill / hook / MCP / CLAUDE.md の並び順
- 自律ループで cache hit 率を **30% → 90%** に上げた実測結果

## エージェントは毎ターン、同じ本を最初から読み直している

脳過敏症の俺は「無駄な刺激＝無駄なトークン」が本当にしんどい。だから自律 brain を組んだとき、最初に気づいたのはこれだった。

skill のロード。hook の発火。MCP の tool 一覧。CLAUDE.md の常駐ルール。人間なら一度読めば終わる内容を、AI は毎ターン初見のように舐め直している。

1 セッション数時間、夜間に走る brain ループ。1 日で数百ターン。同じ system prompt を数百回読ませていたわけだ。考えると、ぞっとした。

## Prompt Caching の正体：4 つの breakpoint と 5 分の TTL

Anthropic の Prompt Caching は、最大 **4 つ** の cache breakpoint を `cache_control` で指定できる。TTL はデフォルト **5 分**（1h も選択可）。キャッシュヒットしたトークンは **入力コストの 1/10**、書き込みは 1.25 倍。

ポイントは「**breakpoint より前が完全一致していれば、そこまでをキャッシュから読む**」という仕様だ。1 バイトでも違えば、その breakpoint は無効化される。

簡単に言うと、キャッシュ境界は「不動のもの」と「揺れるもの」の間に置かないといけない。

## キャッシュを壊す 5 つのアンチパターン

俺が brain 実装で実際にやらかしたやつ、全部晒す。

**①  system prompt 末尾のタイムスタンプ**
「Today is 2026-04-22」を system の末尾に入れていた。これだけで毎日キャッシュ全損。しかも 5 分 TTL なので、実質毎ターン破壊。

**②  MCP ツール定義の順序揺れ**
MCP サーバーを複数繋いでいると、接続順で tools 配列の順序が変わる。JSON としては等価でも、キャッシュキーは文字列一致なので別物扱い。

**③  skill の動的差し込み**
hook で「今このタスクに必要な skill だけロード」とかやると、ロードされる skill セットが毎ターン変わってキャッシュが効かない。

**④  CLAUDE.md への動的変数注入**
「現在の git branch は〜」みたいな変数を CLAUDE.md 相当の system に入れると死ぬ。

**⑤  MCP 再接続による tool 定義の再生成**
long-running セッションで MCP が切れて再接続すると、description の末尾の空白が微妙に変わることがあった。**未検証**だが、これでもキャッシュは外れる。

壊した。壊した。全部壊した。

## 常識崩壊：キャッシュは「最後に置く」ではなく「**動かないものを前に積む**」

よくある誤解。「長い system prompt の最後に `cache_control` を置けば効く」——嘘。

正確には、**breakpoint より前の全トークンが不動** でないと効かない。つまり設計順序がすべて。

俺が落ち着いた並びはこれ:

1. **tools 定義**（MCP 含めて固定順にソート）
2. **system prompt 本体**（CLAUDE.md 由来の不動ルール）
3. `cache_control` breakpoint ①
4. **skill の全文ロード**（動的選択はしない、全部載せる）
5. `cache_control` breakpoint ②
6. **会話履歴**（ここから先は揺れる）
7. 最新ユーザー入力の直前に `cache_control` breakpoint ③

ただし、skill を全部載せるとコンテキストを食うので、プロジェクト単位で skill セットを固定する運用にした。動的に選びたい気持ちはわかる。わかるが、キャッシュ経済性とトレードオフ。

## 実測：cache hit 率 30% → 90% で何が起きたか

brain を 2 週間走らせた実測だ。

**Before（雑実装）**
- cache hit 率: 約 **30%**（自己計測、Anthropic console の usage から逆算）
- 1 ターン平均入力トークン: 22,400
- 初動レイテンシ: **4.0 秒**
- 夜間 8 時間ループの想定 API コスト: 高い（ここは生々しいので伏せる）

**After（並び順を固定化、タイムスタンプを user 側へ移動）**
- cache hit 率: 約 **90%**
- 1 ターン平均課金相当トークン: 実効 2,800 相当
- 初動レイテンシ: **0.6 秒**
- API コスト: **約 1/8**

感動した。静かになった。脳が喜んだ。

正直に言うと、3 割の壁までは誰でも行ける。9 割に載せるのがしんどい。特に hook で system を触る設計にしていると、知らないうちに breakpoint が崩れる。俺は hook を全部 PostToolUse 側に寄せて、system 生成に触らせない運用に変えた。

![]( /images/20260422-1601-claude-code-prompt-caching-api/fig3_before_after.png)

## Claude Code 実装者のためのチェックリスト

自分の brain で使っている規律を置いておく。

- [ ] `CLAUDE.md` に動的変数を入れていないか（日付、branch 名、時刻）
- [ ] MCP tools の順序がセッション間で安定しているか
- [ ] skill はプロジェクト単位で **固定セット** になっているか
- [ ] hook が system prompt を書き換えていないか
- [ ] タイムスタンプは user メッセージ側に置いているか
- [ ] `cache_control` は **tools 直後** と **skill ブロック直後** の 2 点に置いているか

ただし、これは俺の brain 運用での最適解であって、全エージェントに最適とは限らない。短い対話型ボットなら breakpoint 1 個で十分だし、逆に超長文の RAG 系なら 4 個フル活用したほうがいい。

## まとめ：キャッシュは AI の短期記憶の外部化である

人間は同じ資料を朝夕で読み直さない。一度読んだら頭に入っている。エージェントにそれをやらせるのが Prompt Caching だ。

最適化じゃない。設計規律だ。思考の土台を不動のものとして積めているか、という問いに対する答えそのもの。

俺は brain を夜間 8 時間走らせている。キャッシュを整えたら、朝起きたときのログが静かになった。無駄な再読が消えた分、エージェントは本当に考えるべきところに計算を使っていた。

同じことやってる人、いる？ skill や hook のどこに cache breakpoint を置いているか、めちゃくちゃ知りたい。コメントか X で教えてほしい。

---
参考: https://zenn.dev/analysis/articles/thought-analyzer-prompt-caching

#Claude #ClaudeCode #生成AI #AIでやってみた #PromptCaching #MCP #エージェント設計 #自律AI #Anthropic #トークン削減

<!-- figure_style: clean -->
