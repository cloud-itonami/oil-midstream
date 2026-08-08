# oil-midstream

**石油サプライチェーンの *中流*（幹線パイプライン・貯蔵/輸出入ターミナル・その間の
フロー）を扱うと宣言した actor の descriptor と、その書き込みを止める
deny-by-default gate。パイプラインの実データも、それを読むグラフも、ここには無い。
実装ではない。**

`oil-*` は 7 本ある。うち 6 本（`upstream` / **`midstream`** / `refining` /
`trading` / `shipping` / `distribution`）が segment の実体を扱う側で、`oil-coverage`
だけがそれらを上から測る meta actor。**この repo は 6 本のうち、井戸元と製油所の
あいだを結ぶ区間**を担当すると宣言している。

| | ここにあるか |
|---|---|
| actor が**何を名乗り、何を要求し、どの pipeline を持つと宣言しているか** | **ある**（`actor-manifest.jsonld` 8,653 B / `.well-known/did.json` 736 B） |
| **gate**（attestation が 7 つ揃わなければ effect を 1 つも出さない判断） | **ある**（`src/oil_midstream/murakumo.cljc`、226 行） |
| パイプラインを数えるグラフ、cron を撃つ scheduler、XRPC を受ける server | **無い** |
| 幹線パイプライン・ターミナルの実データ | **無い** |

**ここには動くサービスは無い。** `cell-plan` が返すのは「書くとしたら何をどこに
書くか」という**計画**であって、書き込みそのものではない。`:effects` は
`{:op :mst/put-record ...}` という data であり、それを実行する者はこの repo に居ない。

経緯は [docs/adr/0001-descriptor-snapshot-not-an-executor.md](docs/adr/0001-descriptor-snapshot-not-an-executor.md)。
手順は [docs/operator-quickstart.md](docs/operator-quickstart.md)。

## 名前が 2 つあり、解決するのは片方だけ

この repo は自分を 2 通りに名乗っている。**同じ actor の別表記ではなく、
一方は DNS に存在しない。**

| 出所 | 名乗り | 2026-08-09 実測 |
|---|---|---|
| `actor-manifest.jsonld` の `@id`<br>`src/oil_midstream/murakumo.cljc` の `actor-did` | `did:web:oil-midstream.etzhayyim.com` | **解決しない**。`oil-midstream.etzhayyim.com` に A/AAAA レコードが無く、`curl` は `000`（接続前に失敗） |
| `.well-known/did.json` の `id` | `did:web:etzhayyim.com:actor:oil-midstream` | **解決する**。`https://etzhayyim.com/actor/oil-midstream/did.json` が `200` |

**gate が名乗るのは解決しない方**である（`murakumo.cljc:6`）。effect の
`:actor` フィールドに載るのもそちら。これは既知の不整合で、直していない —
どちらを正とするかは etzhayyim 側の identity 決定であって、この snapshot が
勝手に決めてよいことではない。

### この repo の `.well-known/did.json` は配信されていない

`.nojekyll` が置かれているので GitHub Pages 配信を意図していたと読めるが、
2026-08-09 時点で `etzhayyim.github.io/com-etzhayyim-oil-midstream` も
`cloud-itonami.github.io/oil-midstream` も **404**。

さらに、解決する方の URL が実際に返す文書は、**この repo の中の did.json とは
別物**である（`jq -S` して `diff` すると 5 か所相違、`diff` は exit 1）:

| | repo の `.well-known/did.json` | live（`etzhayyim.com/actor/oil-midstream/did.json`） |
|---|---|---|
| crypto suite context | `ed25519-2020` | `jws-2020` |
| `alsoKnownAs` | 4 件（at:// · github · rad: · github.io） | **空配列** |
| `verificationMethod` | **フィールド自体が無い** | 空配列（`_meta` に「ERC725 mirror pending」と注記） |
| PDS endpoint | `https://pds.etzhayyim.com` → **530**（`/xrpc/_health`） | `https://pds.aozora.app` → `/xrpc/_health` が **200** |
| 2 つ目の service | `#aozora`（AozoraAppView、`https://aozora.app`） | `#xrpc-libp2p`（`/dnsaddr/etzhayyim.com/p2p/12D3KooW…`） |

つまり **repo の did.json は live の写しではなく、live より古い（あるいは別系統の）
文書**。ここを「配信されている DID document」として読まないこと。

### live の DID document は、manifest ではなく **gate** の語彙を支持している

live 側だけが持つ `_meta.primaryLexicon` は **`com.etzhayyim.oil-midstream`** であり、
これは `murakumo.cljc` の `collection` 関数が組み立てる綴りと**一致する**。
manifest 側の `com.etzhayyim.apps.oilMidstream.*` ではない。7 本すべて同じ:

```
oil-midstream      com.etzhayyim.oil-midstream
oil-distribution   com.etzhayyim.oil-distribution
oil-upstream       com.etzhayyim.oil-upstream
…（7/7 が com.etzhayyim.<repo 名>）
```

後述する「gate と manifest の語彙が交わらない」問題は、したがって
**gate 側だけの scaffold 由来の綴り崩れ**とは読めない。第三者に相当する live の
identity 文書が支持しているのは gate 側で、`apps.` 名前空間を裏書きするものは
（実測した範囲では）どこにも無い。**どちらが正かはこの snapshot が決めない。**

## 何を扱うと宣言しているか

`actor-manifest.jsonld` は **8 pipeline** を宣言する（cron 2 / subscribeRepos 1 / xrpc 5）。

| trigger | 何を宣言しているか |
|---|---|
| cron `0 */8 * * *`（5 step） | **報告する。** 国別のパイプライン本数と容量(bpd) → ターミナル種別ごとの数 → `constrainedBy` エッジの制約種別ごとの数 を数え、`agent.chat` に要約させ、`derive:social` で社会面に 1 本流す |
| cron `0 */6 * * *`（3 step） | **自分の被覆率を書き戻す。** 自 DID のノード総数と collection 別内訳を数え、`ActorCoverageSnapshot` を MERGE する |
| subscribeRepos `oilShipping.cargo`（1 step） | cargo が来たら、その `loadPort` / `dischargePort` の LOCODE に一致するターミナルを引く |
| xrpc `…infrastructure.getPipeline` | 1 本のパイプラインを返す |
| xrpc `…infrastructure.listPipelines` | パイプラインを最大 50 件返す |
| xrpc `…infrastructure.listTerminals` | ターミナルを最大 50 件返す |
| xrpc `…infrastructure.getTerminalNetwork` | LOCODE 指定で、そのターミナルへ `flowsTo` するエッジを最大 50 件返す |
| xrpc `…health` | パイプライン総数を 1 行返す |

宣言された 4 つの sub-actor（`actors[]`）は、**4 つとも対応する step がある**:
`registry:pipeline`（getPipeline / listPipelines / health）/
`registry:terminal`（listTerminals / cargo 購読）/
`network:flow`（getTerminalNetwork の `flowsTo`）/
`risk:constraints`（8 時間 cron の `constrainedBy` 集計）。

読むグラフラベルは 2 つ（`OilPipeline` を 4 か所、`OilTerminal` を 4 か所）、
エッジ型が 2 つ（`flowsTo` / `constrainedBy`）。書くのは `ActorCoverageSnapshot`
1 か所。使う `fn` は `graph.query` 11 / `graph.write` 1 / `agent.chat` 1 /
`derive:social` 1 で、**宣言された 5 capability のうち `agent.invoke` はどの step でも
使われていない。**

## 購読は 5 件、そのうち handler があるのは 1 件

`triggers.subscribeRepos.collections` は 5 collection を購読すると宣言するが、
`trigger.type == "subscribeRepos"` の pipeline は **1 本しか無い**（`oilShipping.cargo`）。
残り 4 件は、受け取っても実行するものが無い。これは 7 本共通の形である:

| repo | 宣言した購読 | handler pipeline |
|---|---|---|
| oil-upstream | 5 | 1 |
| **oil-midstream** | **5** | **1** |
| oil-refining | 5 | 1 |
| oil-trading | 4 | 1 |
| oil-shipping | 6 | 1 |
| oil-distribution | 4 | 1 |
| oil-coverage | **12** | **0** |

### そして、購読先を書く者もフリート内に居ない

cloud-itonami に `actor-manifest.jsonld` を持つ repo は **39 本**あり、その中に
`graph.write` step は **70 個**ある。この 70 個を全部並べても、
`OilPipeline` / `OilTerminal` / `flowsTo` / `constrainedBy` を**書くものは 1 つも無い**
（読む側は 11 か所 — oil-midstream 8 / oil-coverage 2 / oil-shipping 1）。

購読先 5 件それぞれの供給元:

| 購読先 collection | フリート内で言及する repo | producer か |
|---|---|---|
| `…apps.oilMidstream.pipeline` | この repo のみ（購読側） | **無い** |
| `…apps.oilMidstream.terminal` | この repo のみ（購読側） | **無い** |
| `…apps.oilMidstream.flow` | この repo と `oil-upstream`（両方とも購読側） | **無い** |
| `…apps.oilShipping.cargo` | oil-* 5 本（すべて購読側） | **無い** |
| `…apps.port.portCallEvent` | この repo・`oil-shipping`・**`port-actor`** | **後述。実質は無い** |

**5 件目だけは一見つながっている。** `port-actor` は `graph.write` を 2 つ持ち、
片方が `MERGE (pce:PortCallEvent …) SET … pce.collection =
'com.etzhayyim.apps.port.portCallEvent'` を書く。しかしこれは **グラフのノード**への
書き込みであって、PDS の repo collection への record 公開ではない —— `subscribeRepos`
が運ぶのは後者である。しかも `port-actor` のその pipeline 自体が
`subscribeRepos com.etzhayyim.apps.vessel.portCall` を trigger にしており、
**その `vessel.portCall` を書く graph.write もフリート内に無い**。

```
vessel.portCall（producer 無し）
  → port-actor が購読 → グラフに PortCallEvent ノードを MERGE（record 公開ではない）
  → oil-midstream が port.portCallEvent を購読（届くものが無い）
```

つまり oil-distribution より鎖は 1 本長いが、**端は同じくフリートの外で切れている**。

`oil-coverage` はこの repo の `…apps.oilMidstream.coverageSnapshot` を購読しており、
**測られる側**としての接続だけが両方向で噛み合っている。

## 兄弟 6 本との違いはどこか

`actors=4` / `pipelines=8` は 6 本すべて同じ（`oil-coverage` だけ `actors=6` /
`pipelines=5` で形からして別物）。**購読数と `nanoid` は違う**:

| repo | nanoid | 宣言した購読 |
|---|---|---|
| oil-upstream | `01lupstr` | 5 |
| **oil-midstream** | **`01lm1dst`** | **5** |
| oil-refining | `01lr3f1n` | 5 |
| oil-trading | `01ltrad3` | 4 |
| oil-shipping | `01l5h1p0` | 6 |
| oil-distribution | `01ld1str` | 4 |
| oil-coverage | `011c0v3r` | 12 |

購読数が違うので、後述の gate cell 数も repo ごとに違う（この repo は 16、
oil-distribution は 15）。

## gate は何を止めるか

`src/oil_midstream/murakumo.cljc` は **16 cell × 7 gate** の deny-by-default。
7 つの attestation が 1 つでも欠けると `:status :blocked` で `:effects` は空になる
（実測: 6/7 揃えても `:blocked`、`all-cell-plans` は 16 cell 全部 blocked で総 effect 数 0。
7/7 揃えると 16 cell すべて `:ready` で effect 16）。

7 gate: `:council-charter-attestation` `:no-platform-held-key-baseline`
`:no-probing-baseline` `:murakumo-only-inference-baseline`
`:did-primary-baseline` `:append-only-gate-baseline`
`:kotoba-only-substrate-baseline`

16 という数は manifest から導ける: **5 xrpc + 2 `requiredCollections` +
4 `requiredLoops` + 5 `subscribeRepos` = 16**。

実行して確かめられる（[quickstart](docs/operator-quickstart.md) の手順 4）。

### ⚠ gate が書く collection は、manifest が宣言する collection と 1 つも一致しない

`murakumo.cljc` の `collection` 関数は `com.etzhayyim.oil-midstream.<name>` を
組み立てる。manifest 側の語彙は `com.etzhayyim.apps.oilMidstream.<name>` である。
`apps.` の有無と segment のハイフン/キャメルが違い、**16 個の交わりは空**:

```
gate     com.etzhayyim.oil-midstream.pipeline
manifest com.etzhayyim.apps.oilMidstream.pipeline
```

（上で見たとおり、live の DID document が支持しているのは **gate 側**の綴りである。）

さらに gate は 2 種類の「読むもの」を書き込み先に変えている:

- **XRPC の *メソッド* 名を collection として扱う。** `getpipeline` は manifest では
  「読む API」だが、gate では `com.etzhayyim.oil-midstream.getpipeline` へ
  `:mst/put-record` する計画になる。
- **他 actor の collection を自分の名前空間に付け替える。** manifest が購読する
  `com.etzhayyim.apps.oilShipping.cargo` と `com.etzhayyim.apps.port.portCallEvent` は、
  gate では `com.etzhayyim.oil-midstream.cargo` / `….portcallevent` という
  **自分が書く先**になる。

`requiredLoops` の 4 つ（`shinka` / `koji` / `kyumei` / `domain-knowledge`）も
同じ規則で collection 化されている。いずれも scaffold 生成器の産物と読めるが、
正しい語彙を決めるのは lexicon 側の権限なので**直していない**。

⚠ **namespace が `oil_midstream.murakumo`（アンダースコア）である。** Clojure の
慣習では `oil-midstream.murakumo` と書いてファイル側を `oil_midstream/` にする。
現状でも load はできるが、`(require '[oil-midstream.murakumo])` は**通らない** —
`oil_midstream` と綴る必要がある。

## live の PDS には record が 1 件も観測できない（ただし証拠能力は弱い）

live の did.json が指す `pds.aozora.app` に `com.atproto.repo.describeRepo` を投げると
`{"collections":[]}` が返る。**ただしこの endpoint は存在しない DID にも同じ形で
200 を返す**（対照実験: `did:web:etzhayyim.com:actor:not-a-real-actor-xyz` でも
`{"collections":[],"handleIsCorrect":false}`）。`listRecords` も同様に架空 DID で
`{"records":[]}` を返す。

したがってこの観測から言えるのは **「record は 1 件も観測できない」**までで、
「登録済みだが空」と「そもそも未登録」は**この endpoint では区別できない**。
200 を registration の証拠として読まないこと。

## 2026-06-24 の snapshot であること

etzhayyim monorepo の `20-actors/oil-midstream` から descriptor だけを写した
snapshot。`actor-manifest.jsonld` が `runtime: k8s-langserver` /
`edge: sveltekit-proxy` / `legacyExecutionTier: T1` / `heartbeatRequired: true` と
宣言していても、**その runtime も edge も heartbeat もここには無い。**
`complianceDocs` が指す 2 本（`90-docs/rules/compliance/…` / `90-docs/platform/…`）も、
この repo には存在しない**元 monorepo のパス**である。

`actor-manifest.test.ts` は **走らない** — `package.json` も vitest も無く、
`node_modules` も無い。vitest 前提で書かれた 11 の `it(` / 16 の `expect(` が、
実行されないまま置かれている。`.ts` なのでこの workspace の nbb 経路にも載らない。

commit は 4 本だけ:

| commit | 日付 | 内容 |
|---|---|---|
| `15439a3` | 2026-06-24 | snapshot（manifest / did.json / NOTICE / test.ts） |
| `ca8d5fd` | 2026-07-02 | did:web を `etzhayyim.com` scheme へ移行 |
| `6921bbc` | 2026-07-18 | murakumo WIP の rescue（`src/oil_midstream/murakumo.cljc`） |
| `889c000` | 2026-07-27 | 上の rescue branch を main へ merge |

## 既知のギャップ（この反復で埋めていないもの）

- **test が無い。** 上の表の数（pipeline 8 / sub-actor 4 / cell 16 / gate 7 /
  label 2 / 購読 5・handler 1）と 2 つの DID は、いま**この README が主張している
  だけ**で、実体が変わっても赤くならない。`marine-insurance` は
  `test/…/docs_test.cljs` でこれを固定している —— 同じものがここにも要る。
- **west pin が遅れていた。** superproject の pin は `ca8d5fd`（2026-07-02）で、
  `src/oil_midstream/murakumo.cljc` を含む `889c000` を指していなかった。この
  README を書く時点で main に合わせている。
- **identity の不整合を直していない**（2 名の DID、live との 5 か所差分、
  live の `primaryLexicon` と manifest の食い違い）。
- **collection 語彙の不一致を直していない**（gate と manifest で交わり空）。
- **購読 5 件のうち 4 件に handler が無い**ことを直していない。
