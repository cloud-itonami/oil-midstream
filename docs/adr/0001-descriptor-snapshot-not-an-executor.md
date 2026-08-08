# ADR-0001 — この repo は descriptor snapshot であって executor ではない

- status: accepted
- date: 2026-08-09
- 対象: `cloud-itonami/oil-midstream`
- 上位: superproject ADR-2608080000（成熟度を 1 段ずつ上げる loop）/
  ADR-2608052000（7 軸の測り方）/ ADR-2606231200（sovereign actor repo pilot）
- 先例: `cloud-itonami/oil-distribution` の ADR-0001（同じ形の repo、1 周前）

## 文脈

superproject の成熟度 loop（skill `itonami-maturity-improve`）が、この repo を
`axis-docs`（`own = 0.041`、README 0 バイト）で名指しした。同点の兄弟が 5 本並んでおり
（`:ranking-is-flat? true`）、前周は `oil-distribution` を同じ軸で処理している。

この repo は 2026-06-24 に etzhayyim monorepo の `20-actors/oil-midstream` から
**descriptor だけを写した** snapshot として起こされた。commit は 4 本しか無い:

| commit | 日付 | 内容 |
|---|---|---|
| `15439a3` | 2026-06-24 | snapshot（manifest / did.json / NOTICE / test.ts） |
| `ca8d5fd` | 2026-07-02 | did:web を `etzhayyim.com` scheme へ移行 |
| `6921bbc` | 2026-07-18 | murakumo WIP の rescue（`src/oil_midstream/murakumo.cljc`） |
| `889c000` | 2026-07-27 | 上の rescue branch を main へ merge |

## 問題

README が 0 バイトだったため、この repo を読む者は次の 3 つを区別できなかった:

1. **宣言**（manifest が「こう動く」と書いていること）
2. **判断**（gate が実際に実行できること）
3. **実装**（どこにも無いもの）

名前（`oil-midstream`）と manifest の記述（`runtime: k8s-langserver` /
`edge: sveltekit-proxy` / `legacyExecutionTier: T1` / `heartbeatRequired: true` /
cron trigger）は、素直に読むと **1 と 3 を同じもの**に見せる。実際には 3 は
1 バイトも無い。

## 決定

### 1. README は「何が無いか」から書く

`axis-docs` は README のバイト数を測るが、**バイト数を目的にしない**。読み手が
最初に知るべきことは「動くサービスは無い」であり、それを冒頭 5 行に置く。

### 2. 主張はすべて実行可能な手順に還元し、書いたあとに全部踏む

README が出す数と identity の状態は、すべて `docs/operator-quickstart.md` の
7 手順で再現できる形にした。**書いたあとに全手順を verbatim で実行**し、貼った
出力は実行結果の写しにした（手打ちの擬似出力を置かない）。

これは形式ではなく検査として働いた —— 踏んだ結果、購読 handler を出す jq 式が
`.trigger.nsid` を読んでおり、subscribeRepos trigger では `nsid` が `null` で
`.collections` 側に値が入るため **`handler:` としか出ない**ことが判明した
（`.collections` を join する形に修正）。踏まなければ、動かないコマンドを
「実行結果」として貼っていた。

### 3. 先例（oil-distribution）を雛形として使うが、値は必ず測り直す

7 本は同じ scaffold から出ているので構造は似るが、**数は違う**。実測で違った箇所:

| | oil-distribution | **oil-midstream** |
|---|---|---|
| 宣言した購読 | 4 | **5** |
| gate cell 数 | 15 | **16** |
| sub-actor と step の対応 | `risk:inventory` に対応する step が**無い** | **4/4 すべて対応する step がある** |
| グラフラベル | `ProductTerminal` / `WholesaleHub` | `OilPipeline` / `OilTerminal` + エッジ 2 種（`flowsTo` / `constrainedBy`） |
| 購読先の producer | フリート内に 0 | 見かけ上 1（`port-actor`）だが**実質 0**（決定 5） |

先例の文言を写して数だけ差し替えると、この 5 行はすべて間違う。

### 4. 2 つの DID の不整合は「記録するが直さない」

- `actor-manifest.jsonld` の `@id` と `murakumo.cljc` の `actor-did` は
  `did:web:oil-midstream.etzhayyim.com` を名乗るが、**DNS レコードが無く解決しない**
- `.well-known/did.json` の `id` は `did:web:etzhayyim.com:actor:oil-midstream` で、
  **こちらは 200 で解決する**

どちらを正とするかは etzhayyim 側の identity 決定であり、descriptor snapshot が
片方に寄せてよいものではない。**現状を測って書き、直さない。** repo の did.json を
live（5 か所相違）に合わせることもしない —— live 側は `_meta` に「ERC725 mirror
pending」と書いており、まだ動いている最中の文書である。

### 5. 「購読先を書く者が居ない」を、鎖を 1 本追ってから書く

`oil-distribution` の ADR は「7 本の `oil-*` の `graph.write` はすべて自分の
`coverageSnapshot` だけ」と書いた。この repo で同じことを言う前に、**39 本の
actor-manifest 全体**（`graph.write` 70 個）へ範囲を広げて測り直した。結果:

- `OilPipeline` / `OilTerminal` / `flowsTo` / `constrainedBy` を書く step は
  **39 本のどこにも無い**（読む側は 11 か所）
- ただし 5 件目の購読 `com.etzhayyim.apps.port.portCallEvent` については、
  `port-actor` が `MERGE (pce:PortCallEvent …) SET … pce.collection =
  'com.etzhayyim.apps.port.portCallEvent'` を**書いている**

grep だけで止めれば「producer が居る」と書いてしまう。実際には、それは
**グラフのノード**への書き込みであって PDS の repo collection への record 公開では
なく、`subscribeRepos` が運ぶのは後者である。さらにその pipeline 自身が
`subscribeRepos com.etzhayyim.apps.vessel.portCall` で駆動され、**その
`vessel.portCall` を書く者もフリートに居ない**。

```
vessel.portCall（producer 無し）
  → port-actor が購読 → グラフに PortCallEvent ノードを MERGE（record 公開ではない）
  → oil-midstream が port.portCallEvent を購読（届くものが無い）
```

**鎖は 1 本長いが端は同じく外側で切れている**、と書く。これは欠陥の指摘ではなく
**境界の記述**である。

### 6. live の DID document が gate 側の語彙を支持していることを記録する

`murakumo.cljc` の `collection` は `com.etzhayyim.oil-midstream.<name>` を、manifest は
`com.etzhayyim.apps.oilMidstream.<name>` を使い、**16 個の交わりは空**である。

先例はこの不一致を「どちらも scaffold 生成器の産物」と書いたが、この repo で live の
did.json を読んだところ、**live だけが持つ `_meta.primaryLexicon` は
`com.etzhayyim.oil-midstream`**、すなわち **gate 側の綴り**だった。7 本すべてで同じ
（`com.etzhayyim.<repo 名>`）。

したがって「gate 側だけが崩れている」とは読めない。実測した範囲で `apps.` 名前空間を
裏書きするものは無い。**それでもどちらを正とするかは決めない** —— lexicon の決定は
この snapshot の権限ではない。記録に留める。

さらに gate は 2 種類の「読むもの」を書き込み先に変えている（XRPC メソッド名の
collection 化、他 actor の collection の自名前空間への付け替え）。これも記録のみ。

### 7. 「宣言した購読 5 件のうち handler は 1 件」を明記する

`triggers.subscribeRepos.collections` は 5 件だが、`trigger.type ==
"subscribeRepos"` の pipeline は 1 本（`oilShipping.cargo`）しか無い。7 本すべてで
同じ形で、`oil-coverage` は **12 件宣言して handler ゼロ**。

「購読している」という宣言を「受け取ったら動く」と読ませないために書く。

### 8. live PDS の 200 を registration の証拠として使わない

`com.atproto.repo.describeRepo` は `{"collections":[]}` を返すが、**存在しない DID
にも同じ形で 200 を返す**（対照実験を実施）。`listRecords` も架空 DID で
`{"records":[]}`。

したがって言えるのは「record は 1 件も観測できない」までで、「登録済みだが空」と
「未登録」は**この endpoint では区別できない**。観測を書くときは、その観測が何を
区別できないかも一緒に書く。

## 測ったこと（2026-08-09、すべて実測）

| 主張 | 実測 |
|---|---|
| pipeline 数 | 8（cron 2 / subscribeRepos 1 / xrpc 5） |
| sub-actor 数 | 4（`registry:pipeline` / `registry:terminal` / `network:flow` / `risk:constraints`、**4/4 に対応する step がある**） |
| capability 数 | 5（うち `agent.invoke` はどの step でも未使用） |
| 宣言した購読 / handler | **5 / 1** |
| gate cell 数 | 16 = 5 xrpc + 2 requiredCollections + 4 requiredLoops + 5 subscribeRepos |
| gate 数 | 7、deny-by-default（6/7 でも `:blocked`、16 cell 全部 blocked で effect 0。7/7 で 16 ready / effect 16） |
| gate と manifest の collection の交わり | **空**（16 個すべて） |
| グラフラベル / エッジ | `OilPipeline` 4 / `OilTerminal` 4 / `ActorCoverageSnapshot` 1 write、`flowsTo` / `constrainedBy` |
| フリート全体の書き手 | 39 manifest / 70 `graph.write` 中、上記ラベル・エッジを書くもの **0** |
| `oil-midstream.etzhayyim.com` | A/AAAA 無し、curl `000` |
| `etzhayyim.com/actor/oil-midstream/did.json` | `200` |
| repo did.json vs live | 5 か所相違（`diff` exit 1） |
| live の `_meta.primaryLexicon` | `com.etzhayyim.oil-midstream`（= gate 側の綴り。7 本とも同型） |
| `pds.etzhayyim.com/xrpc/_health` | `530` |
| `pds.aozora.app/xrpc/_health` | `200` |
| `describeRepo`（実 DID / 架空 DID） | どちらも `{"collections":[]}` で 200 |
| GitHub Pages（両 org） | `404` |
| `actor-manifest.test.ts` | 11 `it(` / 16 `expect(`、**走らない**（package.json も node_modules も無い） |
| 兄弟の形 | `actors=4` / `pipelines=8` は 6 本共通、購読数は 4〜6 でばらつく（`oil-coverage` のみ 6 / 5 / 12） |

## 結果

- `README.md` / `docs/operator-quickstart.md` / この ADR を追加した。
- superproject の west pin を `ca8d5fd`（2026-07-02）→ `889c000` に進めた。旧 pin は
  2026-07-18 の rescue より前を指しており、**`src/oil_midstream/murakumo.cljc`
  （226 行の gate）が checkout に現れていなかった**。成熟度 scan は pin ではなく
  checkout を読むので、この repo の substrate は 0 と測られていた —— pin を正した
  副産物として、その過小評価も解消される（`axis-substrate` は loop の目標軸ではない）。

## やっていないこと

- **test を書いていない。** README が主張する数（pipeline 8 / sub-actor 4 / cell 16 /
  gate 7 / label 2 / 購読 5・handler 1）と 2 つの DID を実体と突き合わせるものが無く、
  manifest が変わっても README は赤くならない。`marine-insurance` の
  `test/marine_insurance/docs_test.cljs` が先例で、同型が要る。1 反復 1 軸なので
  次周（`axis-test`）に残した。
- identity の不整合（決定 4）、collection 語彙の不一致（決定 6）、購読 handler の
  欠落 4 件（決定 7）は、いずれも記録のみ。
- namespace の `oil_midstream`（アンダースコア）は直していない —— 直すと
  `require` の綴りが変わる。
- `actor-manifest.test.ts` を nbb の `test/` に書き直していない。
- **同型の兄弟 4 本**（`oil-upstream` / `oil-refining` / `oil-trading` /
  `oil-shipping`）は README 0 バイトのまま。今回 1 本だけ。
