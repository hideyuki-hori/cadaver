# cadaver

クロスボーダー送金におけるカオスエンジニアリング。  
分散システム上のStream処理が障害下でどう振る舞うかを検証する。

クロスボーダー送金は複数のサービス・通貨・規制をまたぐ。  
経路上のどこかが落ちたとき、Streamはどうなるか。  
途中で止まるのか、リトライするのか、補償トランザクションが走るのか。  
それを意図的に壊して観察する。

# いまやってること

DIライブラリの検証を行っている。

- [x] shaku
- [x] entrait
- [ ] nject
- [ ] syrette
- [ ] lockjaw

# なぜクロスボーダー送金か

複数サービス・通貨変換・規制チェックをまたぐため、補償トランザクション・冪等性・最終的整合性といった障害パターンが構造的に避けられない。CRUDアプリでは出会えない壊れ方が、最小構成でも自然に発生する。

ただし送金業務の忠実な再現は目的ではない。通貨はJPY/USDだけ、経路は送金元 → 為替 → 受取の3ノード、AML/KYCは単純なフラグで模擬する。複雑さはドメインではなく障害注入の側に寄せる。

# なぜ作るか

Streamがすきだからこそ、壊れるところを知りたい。

# これまで試したこと

cadaverの基盤となるEventSourcing/CQRSの設計パターンを、Rust × KurrentDBで段階的に検証している。

## 1. Kurrent DB

これまでEventSourcingをやってみて、eventを保存するのにschemaはそこまで重要じゃないなと思った。

```json
{
  "kind": "order_created",
  "version": "1",
  "payload": {
    "user_id": 1
  }
}
```

aggregateするときにkindとversionで集め方を分岐する。

これならrdbじゃなくても(なんならtextファイルでも)なんとかなるかと思い、であればEventStoreに特化したDBを試したくなった。
で保存できたらどうとでもなるのでは？EventStoreのDBを試したい。

### Rust × KurrentDB EventSourcing

CLIで家計簿をつけるアプリを作り、EventSourcingの基本を体感した。
「今の状態」を保存するのではなく「起きたこと」を全部保存するアプローチ。
appendだけで既存データには一切触れない。過去のイベントは絶対に消えない。
KurrentDBはストリームのリビジョンによる楽観的ロックを標準サポートしており、Lost Updateを防げる。
subscribeでリアルタイム監視もできる。イベント駆動の仕組みが自然に作れるのがいい。

- [lab-kurrentdb-rust-client (basic)](https://github.com/hideyuki-hori/lab-kurrentdb-rust-client/tree/basic)
- [Rust × KurrentDBでEventSourcingをやってみた](https://zenn.dev/hideyuki_hori/articles/c11c64d9315e19)

### KurrentDB Projection

残高を確認するたびに全イベントを読み直して計算するのがしんどかったので、Projectionを導入。
KurrentDB内蔵のV8エンジンでJavaScriptを常時実行し、集計結果を保持する仕組み。`get_state`で一発取得。
TypeScriptで型安全にProjectionを書いてES5にコンパイルし、Rustに`include_str!()`で埋め込む構成にした。
`emit()`を使った派生イベント生成が面白い。予算超過時にアラートストリームへ自動でイベントを書き込む。

- [lab-kurrentdb-rust-client (projections/kurrentdb)](https://github.com/hideyuki-hori/lab-kurrentdb-rust-client/tree/projections/kurrentdb)
- [家計簿で学ぶ KurrentDB Projection — Rust と TypeScript でサーバサイド常時計算](https://zenn.dev/hideyuki_hori/articles/93923d6cbdcd20)

### KurrentDB + PostgreSQL CQRS Projection

KurrentDBのProjectionは事前に定義した集計しか返せず、アドホックな集計に向かない。
Catch-up SubscriptionでKurrentDBのイベントをPostgreSQLに投影し、SQLで自由に集計できるようにした。
Vertical Slice Architectureで整理し、Value ObjectでAccount/Amount/Category/Descriptionを型保護。
ユースケースの`run.rs`がインフラを知らない構造にし、Resolverで依存を組み立てている。

- [lab-kurrentdb-rust-client (projections/postgres)](https://github.com/hideyuki-hori/lab-kurrentdb-rust-client/tree/projections/postgres)
- [KurrentDB + PostgreSQL — Rust で CQRS Projection](https://zenn.dev/hideyuki_hori/articles/050b8ae00bebeb)

## 2. DI

EventStoreで家計簿を作ったとき、オレオレDIをやってみたが、コンストラクタインジェクションはしんどかった。
自分の設計力が足りてなくて、納得できるものが作れなかった。

設計センスを身につけるためにライブラリを試すことにした。

### shaku v0.6.2

見覚えのある感じ。
そこはかとなくJavaみを感じる。

- [lab-rust-di/shaku-0-6-2](https://github.com/hideyuki-hori/lab-rust-di/tree/main/shaku-0-6-2)
- [Rust の DI を試す / shaku](https://zenn.dev/hideyuki_hori/articles/95fef715b330ae)

### entrait v0.7.1

依存方向の強制の強制ができるという点がものすごく好みだった。

- [lab-rust-di/entrait-0-7-1](https://github.com/hideyuki-hori/lab-rust-di/tree/main/shaku-0-7-1)
- [Rust の DI を試す / entrait](https://zenn.dev/hideyuki_hori/articles/c794dcb1a0207e)
