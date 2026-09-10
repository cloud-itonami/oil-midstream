# operator quickstart

**この repo で「動かせる」ものは 1 つだけ**（手順 4 の gate）。残りは、宣言と
現実がどれだけずれているかを **自分の端末で確かめる**ための手順である。

所要 5 分。必要なのは `git` / `jq` / `curl` / `dig` / `nbb`。
下に貼ってある出力はすべて 2026-08-09 に実際に実行した結果の写しで、手打ちではない。
手順 1・2 の兄弟比較は、7 本が同じ親ディレクトリに checkout されている前提
（superproject の `orgs/cloud-itonami/`）。

---

## 手順 0 — checkout が gate を持っているか確かめる

`src/` が無ければ、west の pin が `889c000` より前を指している。

```bash
ls src/oil_midstream/murakumo.kotoba && git log --oneline -1
```

```
src/oil_midstream/murakumo.kotoba
889c000 Merge pull request #1 from etzhayyim/rescue/murakumo-wip-20260718
```

（この写しは `889c000` 時点のもの。**`889c000` 以降ならどの commit でもよい** ——
この文書自体が入った `cd89736` を含む。1 行目の `src/…` が出ることだけが条件。）

`src/` が無い場合は superproject 側で pin を進める。**west.yml は生成物なので手で
編集せず、サーバ側 single-entry commit を使う**（詳細は skill `west-pin-advance`）:

```bash
nbb --classpath ".:scripts/nbb_compat" scripts/west-pin-put.cljs oil-midstream HEAD --dry-run
nbb --classpath ".:scripts/nbb_compat" scripts/west-pin-put.cljs oil-midstream HEAD
printf '%s\n' oil-midstream | xargs west update --fetch smart
```

（`xargs` は必須。`printf … | west update` は**引数ゼロ = 全 4,100 project 更新**になる。）

---

## 手順 1 — descriptor の形を数える

README の表の数は、すべてこの 1 コマンドから出ている。

```bash
jq -r '{pipelines:(.pipelines|length), actors:(.actors|length),
        capabilities:(.capabilities|length),
        subscribeRepos:(.triggers.subscribeRepos.collections|length),
        nanoid:.nanoid}' actor-manifest.jsonld
```

```json
{
  "pipelines": 8,
  "actors": 4,
  "capabilities": 5,
  "subscribeRepos": 5,
  "nanoid": "01lm1dst"
}
```

兄弟 6 本と比べる。**`actors` / `pipelines` は同じで、購読数と `nanoid` が違う**
（`oil-coverage` だけ形からして別物）:

```bash
cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                  oil-shipping oil-distribution oil-coverage; do
  printf "%-17s " $n
  jq -r '"actors=" + ((.actors|length)|tostring) +
         " pipelines=" + ((.pipelines|length)|tostring) +
         " subs=" + ((.triggers.subscribeRepos.collections|length)|tostring) +
         " nanoid=" + .nanoid' $n/actor-manifest.jsonld
done; cd -
```

```
oil-upstream      actors=4 pipelines=8 subs=5 nanoid=01lupstr
oil-midstream     actors=4 pipelines=8 subs=5 nanoid=01lm1dst
oil-refining      actors=4 pipelines=8 subs=5 nanoid=01lr3f1n
oil-trading       actors=4 pipelines=8 subs=4 nanoid=01ltrad3
oil-shipping      actors=4 pipelines=8 subs=6 nanoid=01l5h1p0
oil-distribution  actors=4 pipelines=8 subs=4 nanoid=01ld1str
oil-coverage      actors=6 pipelines=5 subs=12 nanoid=011c0v3r
```

---

## 手順 2 — trigger と、実際に触るグラフラベルを見る

```bash
jq -r '.pipelines[] | .trigger.type + " " +
       (.trigger.cron // .trigger.nsid // ((.trigger.collections//[])|join(","))) +
       "  steps=" + ((.steps|length)|tostring)' actor-manifest.jsonld
```

```
cron 0 */8 * * *  steps=5
subscribeRepos com.etzhayyim.apps.oilShipping.cargo  steps=1
xrpc com.etzhayyim.apps.oilMidstream.infrastructure.getPipeline  steps=1
xrpc com.etzhayyim.apps.oilMidstream.infrastructure.listPipelines  steps=1
xrpc com.etzhayyim.apps.oilMidstream.infrastructure.listTerminals  steps=1
xrpc com.etzhayyim.apps.oilMidstream.infrastructure.getTerminalNetwork  steps=1
xrpc com.etzhayyim.apps.oilMidstream.health  steps=1
cron 0 */6 * * *  steps=3
```

読み書きするラベルは 3 つしか出てこない:

```bash
jq -r '.pipelines[].steps[] | (.args.sql // .args.template // empty)' actor-manifest.jsonld \
  | grep -oE '\([a-z]+:[A-Za-z]+' | sort | uniq -c | sort -rn
```

```
   4 (t:OilTerminal
   4 (p:OilPipeline
   1 (c:ActorCoverageSnapshot
```

### 購読は 5 件、handler は 1 件

`triggers.subscribeRepos.collections` は 5 件あるのに、それを受ける pipeline は
1 本しか無い:

```bash
jq -r '.triggers.subscribeRepos.collections[]' actor-manifest.jsonld
jq -r '.pipelines[] | select(.trigger.type=="subscribeRepos")
       | "handler: " + ((.trigger.collections//[])|join(","))' actor-manifest.jsonld
```

```
com.etzhayyim.apps.oilMidstream.pipeline
com.etzhayyim.apps.oilMidstream.terminal
com.etzhayyim.apps.oilMidstream.flow
com.etzhayyim.apps.oilShipping.cargo
com.etzhayyim.apps.port.portCallEvent
handler: com.etzhayyim.apps.oilShipping.cargo
```

7 本すべてで同じ形である（`oil-coverage` は 12 件宣言して handler ゼロ）:

```bash
cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                  oil-shipping oil-distribution oil-coverage; do
  printf "%-17s " $n
  jq -r '"declared=" + ((.triggers.subscribeRepos.collections|length)|tostring) +
         " handlers=" + (([.pipelines[]|select(.trigger.type=="subscribeRepos")]|length)|tostring)' \
    $n/actor-manifest.jsonld
done; cd -
```

```
oil-upstream      declared=5 handlers=1
oil-midstream     declared=5 handlers=1
oil-refining      declared=5 handlers=1
oil-trading       declared=4 handlers=1
oil-shipping      declared=6 handlers=1
oil-distribution  declared=4 handlers=1
oil-coverage      declared=12 handlers=0
```

### 読むノードを書く者がフリート内に居ない

cloud-itonami の 39 本の `actor-manifest.jsonld` を全部見て、`OilPipeline` /
`OilTerminal` / `flowsTo` / `constrainedBy` を **書く** step を探す（読む側と対で
数える）:

```bash
cd ..
echo "--- writes ---"
for f in */actor-manifest.jsonld; do
  jq -r --arg r "${f%/actor-manifest.jsonld}" '.pipelines[]?.steps[]?
    | select(.fn=="graph.write")
    | select((.args|tostring)|test("OilPipeline|OilTerminal|flowsTo|constrainedBy"))
    | $r' "$f" 2>/dev/null
done | sort | uniq -c
echo "--- reads ---"
for f in */actor-manifest.jsonld; do
  jq -r --arg r "${f%/actor-manifest.jsonld}" '.pipelines[]?.steps[]?
    | select(.fn=="graph.query")
    | select((.args|tostring)|test("OilPipeline|OilTerminal"))
    | $r' "$f" 2>/dev/null
done | sort | uniq -c
cd -
```

```
--- writes ---
--- reads ---
   2 oil-coverage
   8 oil-midstream
   1 oil-shipping
```

**writes が空**である。ノードは 7 本の外側から入る前提になっており、その供給元は
どの宣言にも書かれていない。

### 5 件目（`port.portCallEvent`）だけは鎖が 1 本長い

```bash
cd .. && grep -l 'com.etzhayyim.apps.port.portCallEvent' */actor-manifest.jsonld \
  | sed 's|/actor-manifest.jsonld||'
jq -r '.pipelines[] | select((.steps|tostring)|test("PortCallEvent"))
       | "trigger: " + .trigger.type + " " + ((.trigger.collections//[])|join(","))' \
  port-actor/actor-manifest.jsonld
for f in */actor-manifest.jsonld; do
  jq -r '.pipelines[]?.steps[]? | select(.fn=="graph.write")
         | select((.args|tostring)|test("vessel.portCall")) | "producer found"' "$f" 2>/dev/null
done | sort -u
cd -
```

```
oil-midstream
oil-shipping
port-actor
trigger: subscribeRepos com.etzhayyim.apps.vessel.portCall
```

3 行目のループは**何も出力しない** —— `port-actor` が `PortCallEvent` を書くのは
**グラフのノード**であって PDS の record ではなく、しかもその pipeline 自身が
`vessel.portCall` の購読で駆動される。そして `vessel.portCall` を書く者もフリートに
居ない。鎖は 1 本長いだけで、端は同じく外側で切れている。

---

## 手順 3 — 2 つの DID のうち、どちらが解決するか確かめる

```bash
dig +short oil-midstream.etzhayyim.com A          # ← 何も返らない
curl -s -o /dev/null -w '%{http_code}\n' https://oil-midstream.etzhayyim.com/.well-known/did.json
curl -s -o /dev/null -w '%{http_code}\n' https://etzhayyim.com/actor/oil-midstream/did.json
```

```
000
200
```

`000` は「HTTP status が無い」＝ 接続の前段で失敗した、という意味である。
**manifest と gate が名乗る方の DID が、この解決しない側。**

repo の did.json が live の写しでないことも見ておく:

```bash
curl -s https://etzhayyim.com/actor/oil-midstream/did.json > /tmp/live-did-midstream.json
diff <(jq -S . .well-known/did.json) <(jq -S . /tmp/live-did-midstream.json) | head -30
```

`ed25519-2020` vs `jws-2020`、`alsoKnownAs` 4 件 vs 空、`verificationMethod` の
有無、PDS の宛先、2 つ目の service —— 5 か所ずれる（`diff` は exit 1）。

**live だけが持つ `_meta.primaryLexicon` に注目する。** これは manifest の
`apps.` 語彙ではなく、gate が組み立てる綴りと一致する:

```bash
for n in oil-midstream oil-distribution oil-upstream oil-refining \
         oil-trading oil-shipping oil-coverage; do
  printf "%-18s " $n
  curl -s --max-time 12 https://etzhayyim.com/actor/$n/did.json \
    | jq -r '._meta.primaryLexicon // "(none)"'
done
```

```
oil-midstream      com.etzhayyim.oil-midstream
oil-distribution   com.etzhayyim.oil-distribution
oil-upstream       com.etzhayyim.oil-upstream
oil-refining       com.etzhayyim.oil-refining
oil-trading        com.etzhayyim.oil-trading
oil-shipping       com.etzhayyim.oil-shipping
oil-coverage       com.etzhayyim.oil-coverage
```

宛先の生死:

```bash
for u in https://pds.etzhayyim.com/xrpc/_health https://pds.aozora.app/xrpc/_health; do
  printf "%-46s -> " $u; curl -s -o /dev/null -w '%{http_code}\n' --max-time 12 $u
done
```

```
https://pds.etzhayyim.com/xrpc/_health         -> 530
https://pds.aozora.app/xrpc/_health            -> 200
```

repo の did.json が指す方（`pds.etzhayyim.com`）が落ちていて、live が指す方
（`pds.aozora.app`）が生きている。GitHub Pages 側は両 org とも 404:

```bash
for u in https://etzhayyim.github.io/com-etzhayyim-oil-midstream/.well-known/did.json \
         https://cloud-itonami.github.io/oil-midstream/.well-known/did.json; do
  printf "%s -> " $u; curl -s -o /dev/null -w '%{http_code}\n' --max-time 12 $u
done
```

---

## 手順 4 — gate を実際に走らせる（この repo で唯一動くもの）

`/tmp/probe-midstream.cljs` を作る:

```clojure
(ns probe-midstream (:require [oil_midstream.murakumo :as m]))

(def all-gates (set m/common-gates))

(defn summarise [label atts]
  (let [p (m/cell-plan :getpipeline
                       {:attestations atts :request-id "probe-1"
                        :computed-at "2026-08-09T00:00:00Z"})]
    (println (str label "  status=" (:status p)
                  "  effects=" (count (:effects p))
                  "  missing=" (count (:missing-gates p))))))

(println "cells:" (count m/cell-specs) " gates:" (count m/common-gates))
(summarise "none      " #{})
(summarise "6-of-7    " (disj all-gates :kotoba-only-substrate-baseline))
(summarise "all 7     " all-gates)
(println "gate collection:" (m/collection "pipeline"))
(println "actor-did:" m/actor-did)
(let [plans (m/all-cell-plans {:attestations #{}})]
  (println "blocked:" (count (filter #(= :blocked (:status %)) (vals plans)))
           "/" (count plans)
           " total effects:" (reduce + 0 (map #(count (:effects %)) (vals plans)))))
(let [plans (m/all-cell-plans {:attestations all-gates})]
  (println "all-7 ready:" (count (filter #(= :ready (:status %)) (vals plans)))
           "/" (count plans)
           " total effects:" (reduce + 0 (map #(count (:effects %)) (vals plans)))))
```

```bash
nbb --classpath "src:/tmp" /tmp/probe-midstream.cljs
```

```
cells: 16  gates: 7
none        status=:blocked  effects=0  missing=7
6-of-7      status=:blocked  effects=0  missing=1
all 7       status=:ready  effects=1  missing=0
gate collection: com.etzhayyim.oil-midstream.pipeline
actor-did: did:web:oil-midstream.etzhayyim.com
blocked: 16 / 16  total effects: 0
all-7 ready: 16 / 16  total effects: 16
```

**確かめるべきは「6 つ揃えても通らない」ことである** —— 部分的な attestation で
effect が 1 つでも出るなら、それは deny-by-default ではない。

`gate collection:` の行と、manifest 側の綴りを見比べる。**交わりが無い**:

```bash
jq -r '.triggers.subscribeRepos.collections[]' actor-manifest.jsonld
```

```
com.etzhayyim.apps.oilMidstream.pipeline
com.etzhayyim.apps.oilMidstream.terminal
com.etzhayyim.apps.oilMidstream.flow
com.etzhayyim.apps.oilShipping.cargo
com.etzhayyim.apps.port.portCallEvent
```

16 cell の全 collection を出すと、**他 actor の collection まで自分の名前空間に
付け替わっている**ことが見える（`cargo` / `portcallevent`）:

```bash
cat > /tmp/probe-cols.cljs <<'EOF'
(ns probe-cols (:require [oil_midstream.murakumo :as m]))
(doseq [[k v] (sort-by key m/cell-specs)]
  (println (str (name k) "\t" (first (:collections v)))))
EOF
nbb --classpath "src:/tmp" /tmp/probe-cols.cljs
```

```
cargo	com.etzhayyim.oil-midstream.cargo
domain-knowledge	com.etzhayyim.oil-midstream.domain-knowledge
flow	com.etzhayyim.oil-midstream.flow
getpipeline	com.etzhayyim.oil-midstream.getpipeline
getterminalnetwork	com.etzhayyim.oil-midstream.getterminalnetwork
health	com.etzhayyim.oil-midstream.health
koji	com.etzhayyim.oil-midstream.koji
kyumei	com.etzhayyim.oil-midstream.kyumei
listpipelines	com.etzhayyim.oil-midstream.listpipelines
listterminals	com.etzhayyim.oil-midstream.listterminals
pipeline	com.etzhayyim.oil-midstream.pipeline
portcallevent	com.etzhayyim.oil-midstream.portcallevent
shinka	com.etzhayyim.oil-midstream.shinka
shinkaevolution	com.etzhayyim.oil-midstream.shinkaevolution
shinkaknowledge	com.etzhayyim.oil-midstream.shinkaknowledge
terminal	com.etzhayyim.oil-midstream.terminal
```

cell が 16 ある理由も manifest から導ける:

```bash
jq -r '((.pipelines|map(select(.trigger.type=="xrpc"))|length) + (.requiredCollections|length)
        + (.requiredLoops|length) + (.triggers.subscribeRepos.collections|length))' actor-manifest.jsonld
```

```
16
```

---

## 手順 5 — live の PDS に record があるか（そして、なぜこれが弱い証拠か）

```bash
DID="did:web:etzhayyim.com:actor:oil-midstream"
curl -s --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.describeRepo?repo=$DID"
```

```json
{"did":"did:web:etzhayyim.com:actor:oil-midstream","handle":"handle.invalid","collections":[],"handleIsCorrect":false}
```

**この 200 を「登録されている」と読んではいけない。** 存在しない DID でも同じ形が
返る:

```bash
curl -s --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.describeRepo?repo=did:web:etzhayyim.com:actor:not-a-real-actor-xyz"
```

```json
{"did":"did:web:etzhayyim.com:actor:not-a-real-actor-xyz","handle":"handle.invalid","collections":[],"handleIsCorrect":false}
```

`listRecords` も同じで、架空 DID・gate 語彙・manifest 語彙のいずれでも
`{"records":[]}` が返る:

```bash
for c in com.etzhayyim.apps.oilMidstream.pipeline com.etzhayyim.oil-midstream.pipeline; do
  printf "%-42s " $c
  curl -s --max-time 12 "https://pds.aozora.app/xrpc/com.atproto.repo.listRecords?repo=$DID&collection=$c"
  echo
done
```

```
com.etzhayyim.apps.oilMidstream.pipeline   {"records":[]}
com.etzhayyim.oil-midstream.pipeline       {"records":[]}
```

言えるのは **「record は 1 件も観測できない」**までで、「登録済みだが空」と
「未登録」はこの endpoint では区別できない。

---

## 手順 6 — `.ts` テストが走らないことを確かめる

```bash
ls package.json node_modules
```

```
ls: node_modules: No such file or directory
ls: package.json: No such file or directory
```

`actor-manifest.test.ts` は vitest を import しているが、その vitest がここには
無い。**この 11 の `it(` は一度も実行されていない。**

```bash
grep -c 'it(' actor-manifest.test.ts     # 11
```

---

## ここから先に進みたい場合

この repo を「動かす」には、少なくとも次の 5 つが repo の外に要る:

1. `OilPipeline` / `OilTerminal` と `flowsTo` / `constrainedBy` を持つグラフ
   （**そのノードとエッジを書く者はフリート 39 本のどこにも居ない**、手順 2）
2. cron を撃つ scheduler と XRPC を受ける server（`runtime: k8s-langserver`）
3. 7 つの attestation を発行する主体（無ければ gate は永久に `:blocked`）
4. `com.etzhayyim.apps.oilMidstream.*` と `com.etzhayyim.oil-midstream.*` の
   どちらを lexicon の正とするかの決定（live の DID document は後者を支持している、手順 3）
5. 宣言した 5 件の購読のうち、handler を持たない 4 件の受け口（手順 2）

いずれもこの repo は持っていないし、持っていると主張してもいない。
