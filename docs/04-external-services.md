# 外部サービス調査

調査日: 2026-08-13、**2026-08-14 改訂**（クラウド契約 API v2 の発見と実挙動検証を反映）。
**一次情報が確認できた事実**と**未確定事項**を分けて記載する。未確定事項は Phase 0 で潰す。

> 記法: 【実測】は実際にリクエストを送って確認した挙動、【文書】は公式ドキュメントの記載。
> 【文書】のうち実データで未確認のものは、その旨を明記する。

---

## 1. マネーフォワード クラウド契約（電子契約）

### 確認できた事実

- 提供機能（[機能ページ](https://biz.moneyforward.com/contract/function/)より）
  - 電子署名・タイムスタンプ、**すべての電子契約に10年間の長期署名**
  - **合意締結証明書**の発行
  - **関連書類の添付**（契約締結に必要な補足書類を相手方に送付できる）
  - 多者間契約（3者以上）、複数契約の同時締結、一括送信
  - アクセスキー発行（契約内容確認用）
  - スマホからの確認・署名
  - 相手方への署名依頼の英語対応
  - IP制限、SSO、2段階認証、監査ログ
- 料金（[料金ページ](https://biz.moneyforward.com/contract/price/)より）
  - **契約書の送信件数・保管件数による従量課金や上限はない**。利用者数ベース＋初期費用。
  - 「ひとり法人プラン」「スモールビジネスプラン」「ビジネスプラン」では
    **契約書の送信・締結のみ**が利用可能。
  - **「社内申請ワークフローなど、その他の機能を利用したい方は担当者へお問い合わせください」**
    と明記されている。**後述のとおり API v2 は申請（ワークフロー）が中核**であるため、
    この一文が API の利用条件に関係する可能性が高い（→ 未確定事項 #1）。

### API は v1 と v2 が併存する

**v2 が公式に公開されており、こちらが本命。** v1 は開発者サイトに掲載されていない。

| | v1 | **v2** |
| --- | --- | --- |
| ベースURL | `https://api.contract.moneyforward.com/v1` | **`https://api.contract.moneyforward.com/v2`** |
| 認証 | `x-email` + `x-token` の2ヘッダ | **`Authorization: Bearer`（OAuth 2.0）** |
| スコープ | — | `mfc/contract/contract.read` / `.write` |
| 公式ドキュメント | なし | [あり](https://developers.biz.moneyforward.com/docs/partner-api/contract) |
| Webhook | なし | **あり** |
| 締結済み契約の取得 | なし | **あり** |
| 締結済PDFのダウンロード | なし | **あり** |
| データモデル | 契約を直接操作 | **申請（application）中心** |

【実測 2026-08-14】認証ヘッダを付けない場合のエラーメッセージが両者で異なり、別系統であることが確認できる。

```
GET /v1/contracts → {"message":"missing x-email in header"}
GET /v2/contracts → {"message":"missing authorization header"}
```

v1 は `Authorization: Bearer` を渡しても `missing x-email in header` を返す。**v1 に OAuth 経路はない。**

> **v1 は設計の前提にしない。** 開発者サイトから辿れず、変更ポリシーもサポート範囲も不明で、
> v2 の公開により旧世代であることが濃厚になった。以降の記述は v2 を前提とする。

### クラウド契約 API v2

仕様: <https://developers.biz.moneyforward.com/docs/partner-api/contract>（Version 2.0.0）

**認証は OAuth 2.0 の認可コードフローのみ。** 仕様書「はじめに」に明記されている。
APIキー方式（`/auth/exchange`）には**対応していない**（→ §1.2）。

#### エンドポイント【文書】

| 分類 | メソッド | パス | 用途 |
| --- | --- | --- | --- |
| 申請 | GET/POST | `/applications` | 申請の一覧取得・作成 |
| 申請 | GET/PUT | `/applications/{id}` | 申請の取得・更新 |
| 申請 | POST | `/applications/{id}/contracts` | 申請に紐づく契約の作成 |
| 申請 | PUT | `/applications/{id}/contracts/{cid}` | 申請内の特定の契約の更新 |
| 申請 | PUT | `.../document` `.../fields` | 書類・契約情報の更新 |
| 申請 | PUT | `.../partner_representatives` | 取引先代表者名の更新 |
| 申請 | PUT | `.../partner_companies` | 取引先企業の更新 |
| 申請 | POST | `/applications/{id}/submit` | **申請の提出（＝署名依頼の送信）** |
| 申請 | POST | `/applications/{id}/remind` | 申請承認のリマインド送信 |
| 申請 | POST | `/applications/{id}/withdraw` | 申請の取下げ |
| 申請 | POST | `/applications/search` | 申請の検索 |
| 申請 | POST | `/applications/with_stamp` | 押印設定付きの申請の作成 |
| 契約 | GET | `/contracts` | **締結済み契約の一覧**（更新日時の降順） |
| 契約 | GET | `/contracts/{id}` | **締結済み契約の取得** |
| 契約 | GET | `/contracts/{id}/document` | **締結済み契約の書面（PDF）のダウンロード** |
| 契約 | GET | `/contracts/{id}/certificate` | 合意締結証明書のダウンロード |
| 契約 | POST | `/contracts/search` | 締結済み契約の検索 |
| Webhook | GET/POST | `/webhooks` | **Webhook の一覧取得・作成** |
| Webhook | GET | `/webhooks/{id}` | Webhook の取得 |
| マスタ | GET | `/contract_types` `/document_types` `/currencies` `/users` `/workflow_templates` | 各マスタの一覧 |

参照系は `mfc/contract/contract.read`、更新系（Webhook 作成を含む）は `mfc/contract/contract.write`。

#### v1 時点の制約のうち3つが解消する

docs/03・adr/0002・adr/0004 は v1 の制約を前提に書かれている。**v2 では前提が変わる。**

| # | v1 の制約 | v2 での状況 | 影響 |
| --- | --- | --- | --- |
| 1 | Webhook がない | **`POST /webhooks` がある** | **締結状況のポーリング設計が不要**。Cloud Scheduler + Cloud Tasks による定期実行をやめられる（→ [03-architecture.md §1](03-architecture.md)） |
| 2 | 締結状況を取得できない | **`GET /contracts` `GET /contracts/{id}` `POST /contracts/search`** | 「合意締結証明書の取得可否で締結を判定する」回避策が不要（→ [adr/0002](adr/0002-esign-provider-abstraction.md)） |
| 3 | 締結済PDF本体を取得できない | **`GET /contracts/{id}/document`** | [adr/0004](adr/0004-delegate-retention-to-moneyforward.md) の根拠の一つが崩れた。**[adr/0006](adr/0006-self-hosted-retention.md) で決着済み** |
| 4 | 関連書類の添付専用エンドポイントがない | 未確認。`.../document` の複数ファイル可否は実データでの検証が必要 | 受領図面は契約書PDFへ結合する方針を当面維持 |
| 5 | 署名画面の制御はできない | 変わらず。リダイレクト指定のパラメータはない | 決済は自サイトで完結させる（→ [02-business-flow.md §2.5](02-business-flow.md)） |
| 6 | 認証が `x-email`（個人）に紐づく | **OAuth では非該当。**「認可は**事業者単位**で行われるため、連携を設定した担当者が**異動・退職してもAPI連携は停止しません**」【文書】 | **API専用ユーザーは不要**。代わりに「不要な連携が残らないよう定期的に棚卸しする」運用を入れる |
| 7 | エラー形式が仕様と実挙動で異なる | v2 も `{"errors":[{type, code, message}]}` 形式【実測】 | エラーハンドリングはこの形式に合わせる |

> **制約3の解消は adr/0004 の根拠を崩したが、結論としては別の理由で決着した。**
> [adr/0005](adr/0005-electronic-signature-requirement.md) で電子契約サービス自体を使わないことになり、
> **MFに預ける対象がなくなった**ため、[adr/0006](adr/0006-self-hosted-retention.md) で
> 「システムは永続保管せず、確定書面を事務所の手元へ引き渡す」と決めた。

#### 締結フロー（v2）

v1 の「契約を直接作る」形から、**申請（application）を組み立てて提出する**形に変わる。

```
申請の作成 (POST /applications)
  → 申請に紐づく契約の作成 (POST /applications/{id}/contracts)
  → 契約書PDFのアップロード (PUT .../document)
  → 契約情報の保存 (PUT .../fields)          ← 電帳法要件の4項目。必須処理
  → 取引先企業・代表者の登録 (PUT .../partner_companies, .../partner_representatives)
  → 申請の提出 (POST /applications/{id}/submit)  ← ここで署名依頼が飛ぶ
  → [発注者がMFの画面で署名]
  → Webhook で締結完了を受信                    ← ポーリング不要
  → 締結済契約の取得 (GET /contracts/{id})
  → 書面・証明書のダウンロード (GET /contracts/{id}/document, /certificate)
```

**ワークフローテンプレートを事前にMF側で作成しておく必要がある**点は v1 と同じと見られる
（`GET /workflow_templates` が存在するため）。実データでの確認は Phase 0。

### 1.2 認証方式は2つあり、クラウド契約は OAuth のみ

マネーフォワード クラウドの API には**APIキー方式**と**OAuth 2.0**があり、
**対応サービスは完全に独立**している（「互換性はありません」と明記）【文書】。

| | APIキー方式 | OAuth 2.0 |
| --- | --- | --- |
| 用途 | AIエージェント・自動化スクリプト | 業務アプリケーション |
| 人の操作 | **不要**（非対話） | **初回の認可操作が必要** |
| 権限の主体 | **発行したユーザーの権限** | **事業者単位**（個人に依存しない） |
| 有効期限 | キーは無期限、交換したJWTが1時間 | アクセストークン1時間＋リフレッシュトークン |
| **クラウド契約** | **非対応** | **対応（唯一の経路）** |

対応サービスの判定方法【文書】:

- APIキー: アプリポータルの「APIキー新規登録／編集」画面の**「利用可能サービス」**の選択肢に出るか
- OAuth: アプリポータルの**ユーザー編集画面の「アプリ連携権限」**の選択肢に出るか

【実測 2026-08-14】当事務所のアプリポータルでは、APIキーの利用可能サービスは
**「事業者情報」「クラウド連結会計」のみ**でクラウド契約は出ない。
一方 OAuth の「アプリ連携権限」には**クラウド契約がある**。文書の記載と一致する。

#### 認可サーバー【実測 2026-08-14】

`https://api.biz.moneyforward.com/.well-known/oauth-authorization-server` が取得できる。

```
issuer                            https://api.biz.moneyforward.com
authorization_endpoint            https://api.biz.moneyforward.com/authorize
token_endpoint                    https://api.biz.moneyforward.com/token
registration_endpoint             https://api.biz.moneyforward.com/register
grant_types_supported             ["authorization_code", "refresh_token"]
response_types_supported          ["code"]
code_challenge_methods_supported  ["S256"]
token_endpoint_auth_methods       ["none", "client_secret_basic", "client_secret_post"]
scopes_supported                  129種（mfc/contract/contract.read, .write を含む）
```

- **`client_credentials` は非対応。** 実測で `400 unsupported_grant_type`。
  完全に無人のサーバ間認証は組めず、**初回は必ず人がブラウザで認可する**。
- **PKCE(S256) を使う。** 機密クライアントでも付ける。
- アプリ登録画面で選べるクライアント認証方式は `CLIENT_SECRET_BASIC` / `CLIENT_SECRET_POST` の2つ。
  メタデータにある `none`（公開クライアント）は**選べない**。
  → **機密クライアント前提**。SPAから直接叩く構成は想定されていない。Cloud Run の
  バックエンドから呼ぶ本システムの構成と合致する。
- リフレッシュトークンのローテーション有無は未確認。**リフレッシュ処理は直列化する**
  （Cloud Run は複数インスタンスで動くため、並行リフレッシュで新旧が競合する事故を防ぐ）。

### 1.3 現在の契約では v2 を一切利用できない（403 で確定）

> **2026-08-16: これはブロッカーではなくなった。**
> [adr/0005](adr/0005-electronic-signature-requirement.md) により**電子署名を用いず電磁的交付を
> 自前で完結させる**ことを決めたため、**クラウド契約を使わない構成が本線になった。**
> 以下は調査結果の記録であり、相手方が電子契約サービスの利用を求める場合に備えて残す。

【実測 2026-08-14】アプリポータルでOAuthアプリを登録し、認可コードフロー（PKCE S256）で
`mfc/contract/contract.read` および `.write` を含むアクセストークンの取得に**成功**した。
しかし v2 の**参照系・書き込み系のすべてが 403 を返す**。

```
参照系
  GET /v2/contract_types /document_types /currencies /users
      /workflow_templates /contracts /webhooks        → すべて 403
書き込み系
  POST /v2/applications                               → 403

同一のアクセストークンで
  GET https://api.biz.moneyforward.com/v2/tenant      → 200（事業者情報を正常に取得）

403 の本文はいずれも
  {"errors":[{"type":"TYPE_FORBIDDEN","code":"CODE_FORBIDDEN","message":"forbidden"}]}
```

**切り分けは完了している。**

| 状態 | レスポンス |
| --- | --- |
| ヘッダなし | `401 TYPE_UNAUTHORIZED / missing authorization header` |
| 無効なトークン | `401 TYPE_UNAUTHORIZED / access token is not active` |
| **有効なトークン** | **`403 TYPE_FORBIDDEN`** |

遮断が**事業者単位**であることは、次の3点から確定できる。

1. **401 ではなく 403。** トークンは有効と認識されている
2. **同じトークンで別サービス（事業者情報）は 200。** OAuth の実装・スコープ・トークンの扱いは正しい
3. **`POST /applications` がボディの中身に関係なく 403。**
   認可が通っていれば必須項目不足で 400 が返るはずで、それがない
   → リクエストがハンドラに到達する前に弾かれている

さらに `GET /currencies`（通貨マスタ。事業者データを含まない静的な一覧）まで 403 であることから、
**エンドポイント単位でも機能単位でもなく、事業者に対する API 提供そのものの単位**で
閉じられていると判断できる。

#### 契約プランとの関係【実測 2026-08-14】

当事務所は**マネーフォワード クラウドのパーソナルプランのみ**を契約しており、
クラウド契約の別途契約はない。クラウド契約の設定画面では次がロック表示になっている。

> ワークフロー / 契約管理項目 / IPアドレス制限 / アラート / 監査ログ / ユーザーグループ

**v2 API は申請（ワークフロー）が中核**であり、エンドポイントの過半が申請系である。
ワークフローがロックされている状態と 403 は符合する。

公開されている料金ページの記載も、API が標準プランに含まれないことを示唆する。

| プラン区分 | 記載内容 |
| --- | --- |
| 個人向け（パーソナル系） | 「**社内申請機能を利用しない事業者様におすすめです**」。機能はテンプレート作成・送信/締結・締結済み契約書の検索/ダウンロード。**API の記載なし** |
| 法人向け（ひとり法人・スモールビジネス・ビジネス） | 「**契約書の送信・締結のみ**がご利用いただけます。社内申請ワークフローなど、その他の機能を利用したい方は担当者へお問い合わせください」 |

加えて、**クラウド契約には API の製品紹介ページが存在しない**
（クラウド請求書API・クラウド経費API には専用ページがある）。

> **「送信・締結のみ可」は画面操作の話であって、API の利用可否とは別軸である。**
> 書き込み系（`POST /applications`）も 403 であったことから、これは実測で裏付けられた。

#### 残る確認事項（優先度: 低）

APIを利用可能にするために何が必要か（プラン／オプション／申し込み）と、その費用。
**本線から外れたため、問い合わせは急がない。**
相手方が電子契約サービスの利用を求めるケースが出てきた時点で確認すればよい。

→ 未確定事項 #1。

### 署名画面と決済の同居について（要件への回答）

**「MFのメールリンク先の画面で決済してもらう」ことはできない。**
署名画面はMFのドメインで提供され、UIの差し込みも遷移先の制御も第三者にはできない。
API にも署名完了後のリダイレクトURLを指定するパラメータは存在しない。

したがって **決済は自サイト内で完結させ、決済（与信確保）を署名依頼送信のトリガーとする**。
→ [02-business-flow.md §2.5](02-business-flow.md)

### 保管の委譲 — **採らないことに決めた**

締結済契約書の永続保管をMFに委ねる方針（[adr/0004](adr/0004-delegate-retention-to-moneyforward.md)）は
**廃止した**。電子署名を用いず自前で交付を完結させる以上、**MFに預ける対象そのものが存在しない**
（→ [adr/0006](adr/0006-self-hosted-retention.md)）。

以下は当時の検討で確認した一次ソースであり、記録として残す。

- 送信料・保管料0円、件数による課金・上限なし（[料金ページ](https://biz.moneyforward.com/contract/price/)）
- すべての電子契約に10年間の長期署名（[機能ページ](https://biz.moneyforward.com/contract/function/)）
- 電子帳簿保存法の「電子取引」に対応。**ただし「契約書名・契約開始日・終了日・契約金額」を
  すべて入力することが条件**（[MF公式サポート](https://biz.moneyforward.com/support/contract/guide/service-guide/g020.html)）。
  「スキャナ保存」には非対応
- 解約すると**無償利用状態に移行し閲覧・出力ができなくなる場合がある**。退会後はダウンロード不可
  （[解約FAQ](https://biz.moneyforward.com/support/plan/faq/te07.html)）

→ これらはクラウド契約を利用する場合の条件であり、**現行方針では適用されない**。
電帳法対応は本システムで行う（→ [03-architecture.md §6](03-architecture.md)）。

### プロバイダ抽象化は維持する（既定の実装が入れ替わった）

`EsignProvider` による抽象化は維持する（→ [adr/0002](adr/0002-esign-provider-abstraction.md)）。
**ただし既定の実装は `SelfHostedProvider` に入れ替わった**（→ [adr/0005](adr/0005-electronic-signature-requirement.md)）。

- **既定は自前実装。** 電子署名を用いず、承諾取得・PDF確定・交付・通知・受領確認まで本システムで行う
- クラウド契約の実装は、相手方が電子契約サービスの利用を求める場合の選択肢として残す（未実装）
- v1 と v2 でデータモデルが根本的に違った（契約直接操作 → 申請中心）。
  同種の変更が今後も起こりうるため、抽象化の価値は下がっていない
- テスト用のフェイク実装が必要

> **抽象化しておいた判断がここで報われた。** 電子契約サービスの利用を前提に設計していた部分を、
> 上位フロー（承諾 → 重説 → 内容確認 → 決済 → 交付 → 締結）を変えずに差し替えられている。

---

## 2. マネーフォワード クラウド請求書 API v3

**公開されており、見積書に対応している。** 仕様: `https://invoice.moneyforward.com/docs/api/v3/index.html`
（OpenAPI 3.1、調査時点 v3.6.0）

- ベースURL: `https://invoice.moneyforward.com/api/v3`
- 認証: OAuth 2.0 認可コードフロー
  - authorize: `https://api.biz.moneyforward.com/authorize`
  - token / refresh: `https://api.biz.moneyforward.com/token`
  - スコープ: `mfc/invoice/data.read`, `mfc/invoice/data.write`

### 見積書（Quote）関連エンドポイント

| メソッド | パス | 用途 |
| --- | --- | --- |
| GET | `/quotes` | 見積一覧 |
| POST | `/quotes` | 見積作成 |
| GET | `/quotes/{quote_id}` | 見積取得 |
| PUT | `/quotes/{quote_id}` | 見積更新 |
| DELETE | `/quotes/{quote_id}` | 見積削除 |
| GET/POST | `/quotes/{quote_id}/items` | 明細の取得・追加 |
| GET/DELETE | `/quotes/{quote_id}/items/{item_id}` | 明細の取得・削除 |
| POST/DELETE | `/quotes/{quote_id}/posting` | 郵送依頼の申込・取消 |
| PUT | `/quotes/{quote_id}/order_status` | 受注ステータスの更新 |
| POST | `/quotes/{quote_id}/convert_to_billing` | **見積 → 請求書へ変換** |

`Quote` スキーマは `pdf_url` を持つ。`quote_number`, `quote_date`, `expired_date`,
`order_status`, `transmit_status`, `posting_status`, `items`, 税区分別の小計・消費税額を保持。

その他、`/partners`（取引先、部門付き）、`/items`（品目）、`/billings`（請求書）、
`/office`、`/sent_histories`（送信履歴・参照のみ）が利用できる。

**注意点**

- **メール送信のエンドポイントは存在しない**（`sent_histories` は履歴の参照のみ）。
  見積書メールは本システムから送る。契約用リンクを本文に差し込む要件とも整合する。
- `convert_to_billing` があるため、**見積 → 契約 → 請求** の一気通貫が将来的に組める。
- v3では過去に一部エンドポイントの提供終了があったため、
  [お知らせ](https://biz.moneyforward.com/support/invoice/news/)を定期的に確認する。
  API変更ポリシーは[こちら](https://developers.biz.moneyforward.com/docs/common/api_change_policy)。
- 【実測 2026-08-14】アプリポータルの「アプリ連携権限」で当事務所にクラウド請求書は
  **選択されていない**。クラウド契約と同様、利用開始前に権限付与が必要。

### 改正法の「報酬の根拠を明らかにする見積書」との関係

改正法で新設される法24条の10は「報酬の算定の根拠を明らかにするための見積書」の作成を
**努力義務**として定める（→ [01-legal-requirements.md §1.5](01-legal-requirements.md)）。
書面契約義務と異なり施行日に間に合わせる必要はなく、システムで強制もしない。
一方で実務上の価値が高いため、機能としては実装する。
MF請求書の見積明細は品目・数量・単価の構造なので、
業務報酬基準の略算法による算定根拠（区分、床面積、業務人・時間数、直接人件費、
経費、技術料等経費、特別経費）は**本システム側で構造化データとして保持し**、
（告示第8号と、耐震診断・耐震改修用の告示第670号で費目構成が異なる点に注意）、
MFへは明細行および備考（`note`）として展開する。
算定根拠の詳細を記した「業務報酬内訳書」は本システムでPDF生成し、見積書に添付する。

---

## 3. Stripe

- **PaymentIntent の manual capture** を既定とする（→ [02-business-flow.md §2.5](02-business-flow.md)）。
  オーソリの有効期間はおおむね7日。期限が近づいた案件の通知を Cloud Scheduler で実装する。
- Webhook（`payment_intent.succeeded`, `payment_intent.canceled`,
  `charge.refunded` 等）は**署名検証必須**。イベントは冪等に処理する（`event.id` で重複排除）。
- 消費者向けの前払い・手付金は特定商取引法の表示義務に関わる。
  「特定商取引法に基づく表記」ページと、決済前の最終確認画面（金額・時期・返金条件の明示）を実装する。
- 金額は**すべて整数（円）**で扱う。浮動小数点を使わない。

## 4. Clerk

- 事務所スタッフのみが対象（→ [03-architecture.md §2](03-architecture.md)）。
- MFA を組織ポリシーで必須化する。
- ユーザーの public metadata に建築士登録番号・資格区分・管理建築士フラグを持たせ、
  重要事項説明の実施記録に利用する。ただし**契約書に印字する値は
  `Contract.terms` にスナップショットする**（マスタ更新で過去契約の再現性が壊れないように）。
- バックエンドでは Clerk のセッショントークンを JWT として検証する。
  検証をミドルウェアに一元化し、ハンドラ側で素の JWT を触らない。

---

## 5. 未確定事項一覧（Phase 0 で解消）

| # | 事項 | 影響範囲 | 確認先 | 状況 |
| --- | --- | --- | --- | --- |
| 1 | **クラウド契約 API v2 の利用条件**: 全エンドポイントが403。必要プラン、別途申し込みの要否、費用 | 電子契約サービスを使う場合のみ | MFクラウド契約サポート | **優先度低。** 本線から外れた（→ [adr/0005](adr/0005-electronic-signature-requirement.md)） |
| 2 | 未締結時に締結済み契約の取得系が何を返すか | 締結完了の判定ロジック | 実挙動の検証 | v2で `GET /contracts` が使えるため**設計上は解消**。実挙動は #1 の後 |
| 3 | 書類の更新に複数ファイルを追加できるか | 受領図面を別ファイル添付にするか、契約書PDFへ結合するか | 実挙動の検証 | 未解決。#1 の後 |
| 4 | 締結済PDF本体の取得手段 | 自システムでの保管方法 | 実挙動の検証 | **解消（`GET /contracts/{id}/document`）。adr/0004 の再検討が必要** |
| 5 | 認証が個人ユーザーに紐づくか（退職・異動時の失効リスク） | API専用ユーザーの要否 | — | **解消。** OAuth の認可は事業者単位で個人に依存しない【文書】 |
| 6 | 署名依頼メールの文面カスタマイズ可否 | 発注者体験 | MF | 未解決 |
| 7 | 現在の契約プランで送信・締結機能が使えるか | 即時 | MF | #1 に統合 |
| 8 | 電子署名法上の署名方式（事業者署名型／当事者署名型） | 法的効力の説明資料 | — | **非該当。** 電子署名を用いない（→ [adr/0005](adr/0005-electronic-signature-requirement.md)） |
| 9 | IT重説（ビデオ通話での免許証提示）の適法性 | 重説フローの実装 | — | **解消。** 国交省が[実施マニュアル](https://www.mlit.go.jp/jutakukentiku/build/jutakukentiku_house_fr_000125.html)を定めて本格運用しており、これに即せば法24条の7第1項の説明として扱われる。遵守6項目は [01-legal-requirements.md §3.3](01-legal-requirements.md) |
| 10 | 電磁的方法の承諾を案件ごとに取るか包括で取るか | 承諾画面の設計 | 顧問弁護士 | 未解決 |
| 11 | 消費者契約における特商法の適用区分（通信販売／訪問販売） | クーリング・オフ、表示義務 | 顧問弁護士 | 未解決 |
| 12 | 改正法の**施行期日政令**（実際の施行日）と、施行に伴う政省令改正の内容 | スケジュールと法定必須項目 | 国土交通省の公布を待つ | 未解決 |
| 13 | 解約・退会時の一括エクスポート手段 | 永続保管をMFに委ねる前提 | — | **消滅。** 保管を自前に戻した（→ [adr/0006](adr/0006-self-hosted-retention.md)） |
| 14 | 電帳法の保存要件の充足 | 保存要件 | 自前対応 | **方針確定。** 保管が事務所の手元に移るため、真実性は**規則4条1項4号（事務処理規程）が唯一の手段**。検索要件は小規模事業者の特例で外れる見込み。→ [03-architecture.md §6](03-architecture.md)。**規程の整備はリリースのブロッカー** |
| 15 | **リフレッシュトークンのローテーション有無と有効期限** | トークン更新処理の排他制御 | 実挙動の検証 | 新規。#1 の後 |
| 16 | **ワークフローテンプレートの事前作成の要否**（v2） | 申請作成の前提条件 | 実挙動の検証 | 新規。#1 の後 |
| 17 | アクセスキーをAPIから指定できるか（v2） | 電子契約サービスを使う場合のみ | 実挙動の検証 | **非該当。** 自前実装では署名付きリンクとワンタイムコードで担保する（→ [03-architecture.md §2](03-architecture.md)） |
| 18 | **国交省の実施マニュアル・技術的助言の改定**（新規） | 改変防止の解釈（→ [01-legal-requirements.md §4.2.3](01-legal-requirements.md)）とIT重説の遵守事項。マニュアルは令和3年9月1日版で改正法より前 | 国土交通省の公表を待つ | 未解決。#12 と連動 |

**解消済み**: 「見積書の規定が努力義務か義務か」は、改正法本文（令和8年法律第74号 第24条の10）で
**努力義務**であることを確認した。監督処分の対象にも含まれない（法26条2項1号の列挙外）。
→ [01-legal-requirements.md §1.5](01-legal-requirements.md)

> **2026-08-16 時点で、クラウド契約 API に依存する未確定事項（1・2・3・15・16・17）は
> すべて優先度が下がった。** 電子署名を用いず自前で交付を完結させる方針に決まったため
> （→ [adr/0005](adr/0005-electronic-signature-requirement.md)）、これらは
> 「相手方が電子契約サービスを求めた場合に備えた調査」の位置づけになる。
>
> **残る本線の未確定事項は 6・10・11・12 と、新設の 18。** いずれも顧問弁護士レビューまたは
> 政省令の公布待ちであり、実装の着手を妨げない。
