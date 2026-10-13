# app-mail-relay

**`relay.itonami.cloud` —— エージェント向けメールの受信箱。** エージェントが
自分の箱を持ち、相手ごとに別の顔で受け取り、要らなくなったら 1 本だけ止める。
**送受信とも Resend backed**、受けた 1 通は edge で封をして R2 に置き、
1 通ごとに検証可能な CID が付く。

```
Cloudflare Email Routing ─┐
                          ├─> admission ─> persona 振り分け ─> 封 ─> R2 + CID
Resend Receiving webhook ─┘                                          │
                                                                     └─> webhook
POST /v1/inboxes/{inbox}/messages ─> persona が From を決める ─> Resend ─> 送信
```

## 何がどこにあるか

判断は全部 `.cljc` で、IO は Worker（CLJS）に閉じている。

| ns | 何を決めるか | 実行時 |
|---|---|---|
| `mail-relay.account` | **1 アカウント**（開く・閉じる・鍵を回す） | 純 |
| `mail-relay.inbound` | **受けた 1 通**（受けるか・どの顔宛か・何を覚えるか） | 純 |
| `mail-relay.send` | **出す 1 通**（出してよいか・どの From か） | 純 |
| `mail-relay.store` | 保管の形（鍵名・平文で残るもの・CID） | 純 |
| `mail-relay.authz` | route の `:auth` を実際の検査へ | 純 |
| `mail-relay.api` | route 表（**データ**。dispatch も公開文書もここから） | 純 |
| `mail-relay.seal` | 封（AES-256-GCM / X25519） | CLJS |
| `mail-relay.resend` | Resend の両側（webhook 検証・送信実行） | CLJS |
| `mail-relay.root-auth` | `did:key` の Ed25519 で署名検証 | CLJS |
| `mail-relay.worker` | 読む・引く・撃つ・書く | CLJS |

作り直していないもの: `kotoba-lang/persona`（1 通ごとの振り分けと返信の From）、
`kotoba-lang/mail`（メールモデル）、`kotoba-lang/mailer`（送信 effect → provider
request）、`kotoba-lang/envelope`（封）、`org-ietf-mime`（MIME）、
`kotoba-lang/word-id`（読み上げられる 4 語）。

## 予算は各 root の糸で数える（サーバに台帳を置かない）

箱を開く・送るたびに、root は自分の**糸**（`kotoba-lang/chain` の commit 連鎖。
1 entry = 1 commit、`actor` が root の DID、`prev` が 1 つ前）に entry を 1 つ
積み、その CID を did:key で署名して依頼に添える（`mail-relay.ito`、
ADR-2610101117 の 3）。

```text
"ito": {"head":   "<いちばん新しい entry の CID>",
        "blocks": {"<cid>": "<dag-cbor の base64url>", …},
        "sigs":   {"<cid>": "<Ed25519 署名の base64url>", …}}

署名する文字列: "mail-relay.ito.v1\n" + cid

entry の state:
  {"type" "invite/issue" "at" ms "invitee" did "inboxes" n "expires" ms}   招く側の糸
  {"type" "inbox/open"   "at" ms "invite" <invite/issue の CID>}            開く側の糸
  {"type" "mail/send"    "at" ms "to" [address …]}                         送る側の糸
```

relay は糸を検証し、規則を最初から再生して判断する。**台帳は持たない。**

- **invite** は招く側（`RELAY_INVITER_ROOTS` に載った DID。空なら誰も招けない）が
  自分の糸に積んだ `invite/issue`。何回使ったかは、招かれた側の糸にある
  `inbox/open` を数えて決まる。
- **送信**は、直近 24 時間に**初めて連絡した相手**の数で上限をかける（既定 20）。
  前に送った相手には何度送っても数えない。1 通の宛先は 10 まで。entry の宛先と
  実際の宛先が違えば断る。
- relay が覚えるのは**最後に見た head** だけ（R2 `ito/<did>`）。同じ head をもう
  一度出すと `stale-head` で断る —— 1 つの entry で 2 回は出せない。
- **分岐**（同じ位置に別々の entry を 2 つ署名した）を見つけたら、2 つの署名を
  確かめて warrant を保管し（R2 `warrant/<did>`）、その root の開設も送信も
  止める。warrant は `GET /v1/ito/{root}` で公開され、他の relay が読める。
- 断られた entry を relay は見ていないので、client はそれを捨てて前の head から
  積み直してよい。見ていない entry は分岐の証拠にならない。

## なぜ鍵ではなく root で認可するのか

2026-08-21、AgentMail 互換の受信箱を作り、返ってきた `account_key` を保管し
損ねた。鍵は一度しか返らないので、その箱は読めなくなった。そこまでは自業自得
だが、問題はその先で —— **解放する経路が無かった**。`DELETE` は 404、鍵が無い
ので認証もできない。死んだ箱が「1 origin につき 1 箱」の枠を占有したまま、
こちらからは何もできなくなった。

ここから 2 つ引いてある。

- **`close` と `rotate` は root（所有者の DID）で認可する。鍵は要らない。**
- **入場制限を課すなら、退場経路を同時に出す。**

`test/mail_relay/account_test.cljk` の `losing-the-key-does-not-strand-the-inbox`
がこの事故の再演になっている。

### root は「DID をヘッダに書く」ではない

DID は**公開の識別子**なので、それで認可すると、DID を知る誰でも他人の箱を
閉じられる。鍵を失った人を救う経路が、そのまま誰でも使える破壊経路になる。

だから root は**署名**で通る。`did:key:z6Mk…` に埋まっている Ed25519 公開鍵で、
この 1 リクエストに対する署名を検証する（`mail-relay.root-auth`）。署名の対象は
method・path・時刻・nonce を**全部**覆う（`mail-relay.authz/challenge-string`）——
1 つでも落とすと、その軸で付け替えか再生ができる。

```
X-Relay-Root:      did:key:z6Mk…
X-Relay-Timestamp: 1787000000
X-Relay-Nonce:     n-1
X-Relay-Signature: <base64url(Ed25519(challenge))>

challenge = "mail-relay.authz.v1\nDELETE\n/v1/inboxes/inbox_a\n1787000000\nn-1"
```

## 平文で何が残るか

`mail-relay.store/clear-fields` がこの設計の正直さの境界。1 行ごとに
**なぜ封じられないか**が書いてある。`subject`・本文・添付はそこに無い ——
暗号文の中にあり、そのことをテストが主張する。

Worker は受取人の**公開鍵しか持たない**ので、自分が保管したものを読めない
（`open` で `pub` を渡さない箱は開かない。渡されなければサービスが鍵を持つ
ことになり、それはこの設計が引き受けないもの）。

### CID は暗号文に対して付く

`:mail/cid` は CIDv1（raw / sha2-256）で、**ネットワークを触らずに出る**。
指しているのは暗号文であって、差出人が送った本文ではない。

平文のダイジェストは**封の中**に入れてある。鍵を持つ側だけが開いて、
CID → 暗号文 → 平文 → そのダイジェスト、と鎖を繋げられる。`cid-covers` を
関数にしてあるのは、「CID があるので本文を証明できる」という嘘を呼び出し側に
一度は聞かせるため。

`:mail/pinned` は別のフィールド。CID は「何を保管したか」、pin は
「kotobase から引けるか」。1 つにまとめると、pin が落ちた 1 通の CID まで消える。

## API

route 表は `mail_relay/api.cljc` に**データとして**1 つだけ持つ。Worker の
dispatch も公開文書（`GET /v1/routes`）もここから出る。
`worker_test.cljs` の `the-route-table-and-the-dispatch-cannot-drift` が
**表の op 集合と handler の鍵集合が等しい**ことを主張する。

```
POST   /v1/inboxes                        :none  入場（root・pub・糸。糸の head が invite を指す）
GET    /v1/ito/{root}                     :none  ← この relay が見た糸の head と warrant
GET    /v1/inboxes/{inbox}                :key
DELETE /v1/inboxes/{inbox}                :root  ← 解放経路
POST   /v1/inboxes/{inbox}/keys/rotate    :root  ← 鍵の回復
GET    /v1/inboxes/{inbox}/messages       :key
GET    /v1/inboxes/{inbox}/messages/{msg} :key   index + 封じた chunk + CID
GET    /v1/inboxes/{inbox}/threads        :key   ← 501。理由は下
POST   /v1/inboxes/{inbox}/messages       :key   送信（糸で予算 → persona → mailer → Resend）
POST   /v1/inboxes/{inbox}/personas       :key   相手ごとの顔
DELETE /v1/personas/{address}             :key   その 1 本だけ止める
GET    /v1/personas/{address}/senders     :key   誰から来たか = 漏洩の帰責
POST   /v1/inboxes/{inbox}/webhooks       :key   届くのは index だけ
```

`:novel` が付いた route は既存サービスに無い面で、なぜ要るかを `:basis` に
書いてある。宣伝ではなく、次の整理で消されないための理由。

### threads が 501 なのは、封をした帰結

スレッドは `References` / `In-Reply-To` から組む。それはヘッダごと封の中に
あり、**サービスは読めない**。読めるようにするには平文で持つしかなく、それは
封をした意味を消す。組むのは鍵を持つ client 側。501 は未実装ではなく、
この設計の帰結として返している。

### 送信で From を受け取らない

`POST .../messages` に `from` は無い。呼び出し側が選ぶのは `address`（どの顔で
出すか）で、実際の From は `persona.relay/outbound` が返す値をそのまま使う。
`from` を渡すと**黙って無視せず 400 で断る** —— 無視すると、From を指定した
つもりの client が指定できていないことを一生知らない。

受信を alias にするのは簡単で、たいていの実装はそこで終わる。壊れるのは返信で、
1 通を本体から返した瞬間に全部の顔が結びつく。しかも相手には普通に届くので
誰も気づかない。

## 使う

```clojure
(require '[mail-relay.account :as account])

(def r (:mail-relay/registry
        (account/open {} {:id "inbox_a" :root "did:key:z6Mk…"
                          :key-digest "sha256:…" :origin "203.0.113.7"
                          :now 1787000000000})))

(account/close r  {:id "inbox_a" :root "did:key:z6Mk…"})   ; 鍵を失っても閉じられる
(account/rotate r {:id "inbox_a" :root "did:key:z6Mk…" :new-key-digest "sha256:…"})
```

## テスト

```bash
npm install
npm test                    # ClojureScript（Worker と同じ経路）— 封も Ed25519 も本物
kbb -M:local:test       # JVM
```

Worker は CLJS で動くので、JVM で通ることは証拠にならない。両方で回す。
判断の層は `.cljc` なので**同じテストが両方で走る**。

- JVM: 45 tests / 199 assertions
- CLJS: 73 tests / 330 assertions（`seal_test` / `resend_test` / `worker_test` は
  CLJS のみ —— Web Crypto に同期 API が無い）

`worker_test.cljs` は Cloudflare の binding だけを偽物にして、開く→顔を発行→
受ける→一覧→誰から来たか→出す、を端から端まで通す。`resend_test.cljs` は
Svix 署名を**本物の HMAC で作って**投げ、通ったものが Email Routing と同じ
`save-raw!` を通ること、再送が 2 通目を作らないことを見る（偽の R2 は
`onlyIf` を実装してある —— 無視する偽物で試すと冪等性のテストが何も主張
しない）。base58 と CID は**外の実装が同意する固定値**で止めてある（空
ファイルの CIDv1、python の base58 で作った `did:key`）—— 自前実装が自分に
同意しても意味がない。

## ビルドとデプロイ

```bash
npm run build               # amu compile --target wasm32-browser worker -> out/worker.js
npx wrangler deploy
```

デプロイの前に要るもの（**まだ入っていない**）:

1. `wrangler.jsonc` の `REPLACE_WITH_KV_NAMESPACE_ID` を実際の KV に
2. R2 bucket `itonami-mail-relay`
3. `wrangler secret put RESEND_API_KEY` / `RESEND_WEBHOOK_SECRET`
4. Cloudflare Email Routing の catch_all をこの Worker に向ける（zone 側の API。
   wrangler は管理しない）
5. Resend で `relay.itonami.cloud` を検証し、DKIM の CNAME を入れる
6. `relay.itonami.cloud` の DNS —— 2026-08-28 時点で A も CNAME も無い

## まだ無いもの

正直に書く。

- **デプロイ**。`relay.itonami.cloud` は未解決のまま。上の 6 つが要る。
- **到達性**（SPF / DKIM / DMARC・レピュテーション）。外部に売るならここが
  開発の本体。
- **relay をまたいだ分岐の検出**。各 relay は自分が見た head しか知らない。
  別の relay に古い位置から分岐した糸を見せると、両方の relay が head を
  突き合わせるまで見つからない。`GET /v1/ito/{root}` はそのための面だが、
  relay どうしが実際に読み合う経路（ADR-2610082200 の 布）はまだ無い。
- **最後に見た head の保管**は R2。消えても正しさは壊れないが、その relay で
  古い head を出し直される余地が増える。
- **PoW・評判・招いた側の予算を減らすこと**はまだ。返信（相手から先に来た）を
  初めての相手として数えないことも、まだ（今は糸に無い相手は全部初めて）。
- **origin あたりの枠は best-effort**。registry を KV に置いており、KV は結果
  整合なので、同一 origin から同時に 2 本開くと両方通りうる。origin は依頼の
  本文の値なので、枠そのものが入場の制限にはならない（入場を絞るのは糸と
  invite）。registry も糸の entry から再構成できる形に移す（ADR-2610101117）。
  特定の基盤の直列化器（Durable Object など）には依存させない。
- **添付の大きい 1 通**。`GET .../messages/{msg}` は封じた chunk を base64 で
  応答に載せる。数 KB のメールなら問題ないが、25 MiB の添付には向かない ——
  chunk を個別に引く route が要る。
- **共有ライブラリ**。Resend の Svix 検証と R2 lease は
  `net-kotobase/control-plane/mail-worker` からの**写し**。次に直す人が 2 箇所
  あることを知るために書いておく。2 つ目の写しが生まれる前に括り出す。

設計判断は superproject の ADR-2608211300。
