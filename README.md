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
| `mail-relay.admission` | SMTP/P2P 共通の deliver/quarantine/reject | 純 |
| `mail-relay.p2p` | 署名対象・expiry・replay を固定する暗号文 envelope | 純 |
| `mail-relay.jmap` | 復号後 mailbox の client-side JMAP projection | 純 |
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
- **返信**も数えない。relay は受けた 1 通ごとに **receipt**（『この差出人が、
  この root に、この CID の 1 通を送った』を relay の did:key で署名したもの）を
  出し、メッセージ一覧の `receipt` に載せる。返信の `mail/send` entry に
  `"receipts" [receipt]` を入れると、その差出人は初めての相手として数えない。
  receipt を偽れるのは relay の鍵を持つ者だけで、受け入れる relay の DID は
  `RELAY_RECEIPT_DIDS` に並べる（自分の DID も入れる）。receipt は依頼で来ても、
  布から来た head に入っていても、署名を確かめてから受け入れる。
  relay の鍵は `RELAY_SIGNING_KEY`（Ed25519 の PKCS#8、base64url）。未設定なら
  receipt を出さず、返信も初めての相手として数える。
- relay が覚えるのは**最後に見た head** だけ（R2 `ito/<did>`）。同じ head をもう
  一度出すと `stale-head` で断る —— 1 つの entry で 2 回は出せない。
- **分岐**（同じ位置に別々の entry を 2 つ署名した）を見つけたら、2 つの署名を
  確かめて warrant を保管し（R2 `warrant/<did>`）、その root の開設も送信も
  止める。warrant は `GET /v1/ito/{root}` で公開され、他の relay が読める。
- 断られた entry を relay は見ていないので、client はそれを捨てて前の head から
  積み直してよい。見ていない entry は分岐の証拠にならない。

## 布 —— relay の網

relay が 1 台なら上で足りる。複数台にすると、別々の relay に別々の分かれ方を
見せる分岐は、各 relay が自分の見た head しか知らない限り見つからない。そこで
relay どうしを `kotoba-lang/nuno`（ADR-2610082200 の 布）でつなぐ。

- `RELAY_SELF`（自分の URL）と `RELAY_PEERS`（全 relay の URL、カンマ区切り）を
  設定する。`RELAY_PEERS` が空なら何もしない。
- head を進めたら、その root の DID に Kademlia の距離で近い 3 台（authority）に
  `POST /v1/nuno/ops` で配る。受けた relay は糸と署名を自分で検証し、伸びて
  いれば保持、分岐していれば warrant を作る。
- warrant は全 relay に流れ、受けた relay は証拠（2 つの entry が本人の鍵で
  署名され、同じ位置に立つこと）を自分で確かめてから保管し、流し直す。偽物は
  流さない。
- 網に流すものはそれ自体で正しさを示せるので、`/v1/nuno/ops` に認証は無い。
  送り手は信用しない。

テストは relay 3 台を同じプロセスに立て、r1 と r2 に別々の分かれ方を見せると
warrant が 3 台すべてに届き、どの relay でもその root が止まることを確かめる。

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
POST   /v1/nuno/ops                       :none  ← 布（relay の網）からの head / warrant
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

## テストと native qualification

旧 `npm test` は kbb/Sci と表示されるが、依存解決で nbb と JVM を起動するため
acceptance gate から退役した。JVM・nbb・shadow-cljs を fallback にしない。

必要な正規 gate は Amu selfhost + Kotoba native であり、forbidden launcher の
negative control を含む。2026-10-25 時点の public native Amu は
`x86_64-linux-static` だけが qualified で、browser/Worker project link は未対応。
したがって、この checkout の source tests は存在するが native qualification は
**blocked**。parser、旧 test、生成済み Worker を代用して green としない。

新しい SMTP/P2P admission、暗号文 envelope、client-side JMAP projection の境界は
[`docs/itonami-mail-mvp.md`](docs/itonami-mail-mvp.md) にまとめてある。

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
- **布の残り**。authority の数は 3 で固定、網のメンバーは `RELAY_PEERS` の
  静的な一覧（kad.table による発見は無い）。head の配布と分岐の判定の間には
  1 往復の窓があり、その間に 2 台の relay がそれぞれ 1 通ずつ出しうる（その後は
  全台で止まる）。`/v1/nuno/ops` は大きな本文や大量の op に対する速度制限を
  持たない。
- **最後に見た head の保管**は R2。消えても正しさは壊れないが、その relay で
  古い head を出し直される余地が増える。
- **admission の host 接続**。純粋な `mail-relay.admission` は PoW/invite/postage、
  rate、domain authentication、任意の humanity score を合成するが、Worker の
  SMTP/P2P intake へはまだ接続していない。receipt に有効期限は無い
  （一度書いてきた人への返信は、いつでも返信として扱う）。
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
