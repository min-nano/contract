# 外部サービス調査

調査日: 2026-08-13。**一次情報が確認できた事実**と**未確定事項**を分けて記載する。
未確定事項は Phase 0 で潰す。

---

## 1. マネーフォワード クラウド契約（電子契約）

### 確認できた事実

- 提供機能（[機能ページ](https://biz.moneyforward.com/contract/function/)より）
  - 電子署名・タイムスタンプ、**すべての電子契約に10年間の長期署名**
  - **合意締結証明書**の発行
  - **関連書類の添付**（契約締結に必要な補足書類を相手方に送付できる）
    → ただし**これは画面上の機能**であり、APIに対応するエンドポイントは見当たらない（下記の制約4）
  - 多者間契約（3者以上）、複数契約の同時締結、一括送信
  - アクセスキー発行（契約内容確認用）
  - スマホからの確認・署名
  - 相手方への署名依頼の英語対応
  - IP制限、SSO、2段階認証、監査ログ
- 料金（[料金ページ](https://biz.moneyforward.com/contract/price/)より）
  - **契約書の送信件数・保管件数による従量課金や上限はない**。利用者数ベース＋初期費用。
  - 「ひとり法人プラン」「スモールビジネスプラン」「ビジネスプラン」では
    **契約書の送信・締結のみ**が利用可能。ワークフロー等は別途問い合わせ。

### クラウド契約 API v1（発見・検証済み）

**API は存在する。** 仕様: <https://api.contract.moneyforward.com/v1/docs/index.html>
（Swagger 2.0 / `doc.json` / タイトル「マネーフォワード クラウド契約 API」version 1.0）

> 開発者サイト（developers.biz.moneyforward.com）の API リファレンスには掲載されておらず、
> 各製品サイトの「開発者の方」欄にも記載がない。**製品ドキュメントから辿れない位置にある**ため、
> 提供条件・必要プラン・サポート範囲は Phase 0 でマネーフォワードに確認すること。

- ベースURL: `https://api.contract.moneyforward.com/v1`
- **認証: `x-email` と `x-token` の2ヘッダ**（OAuth ではない）
  - 未指定時: `401 {"errors":[{"type":"TYPE_UNAUTHORIZED","code":"CODE_INTERNAL_PARTNER_UNAUTHORIZED","message":"missing x-email in header"}]}`
  - `x-email` のみ: `missing x-token in header`
  - 両方指定・無効なトークン: `access token is not active`
  - → 実挙動で確認済み（2026-08-13）。トークンの発行方法は Phase 0 で確認する。

#### エンドポイント

| メソッド | パス | 用途 |
| --- | --- | --- |
| GET | `/contract_types` | 契約種別一覧 |
| GET | `/contract_templates` | 契約テンプレート一覧 |
| GET | `/workflow_templates` | ワークフローテンプレート一覧 |
| GET | `/document_types` | 書類種別一覧 |
| GET | `/users` | ユーザー一覧（契約担当者の指定に使う） |
| GET | `/currencies` | 通貨一覧 |
| POST | `/contracts` | **下書き契約の作成** |
| POST | `/contracts/with_template` | テンプレートから下書き契約を作成 |
| GET | `/contracts` | 下書き契約の一覧（**下書きのみ**） |
| GET/PUT | `/contracts/{id}` | 下書き契約の取得・更新（**下書きのみ**） |
| POST | `/contracts/{id}/documents` | **契約書PDFのアップロード**（multipart/form-data） |
| PUT | `/contracts/{id}/fields` | 契約情報（管理項目）の保存 |
| POST | `/contracts/{id}/partner_companies` | **相手方と承認者（氏名・メール・会社名・言語・アクセスキー）の登録** |
| POST | `/contracts/{id}/confirm` | **送信（＝承認者への署名依頼メール送信）** |
| POST | `/contracts/{id}/remind` | **署名依頼のリマインド送信** |
| POST | `/contracts/{id}/withdraw` | 取下げ（`comment` 必須） |
| GET | `/contracts/{id}/certificate` | **合意締結証明書のダウンロード**（`application/octet-stream`） |
| — | `/multiple_contracts/...` | 複数契約の一括作成・送信・締結（同等の操作群） |

`POST /contracts` の必須項目は `contract_type_id`, `name`, `person_in_charge_id`, `workflow_template_id`。
**ワークフローテンプレートを事前にMF側で作成しておく必要がある。**

`payload.Approver` の必須項目は `email`, `locale`, `name`。任意で `company_name`, `access_key`。
→ **アクセスキーをAPIから指定できる**ため、発注者への追加認証として自システムで発行した値を渡せる。

#### これで実現できること

Phase 1 の締結フローを**完全に自動化できる**。

```
下書き作成 (POST /contracts)
  → 契約書PDFアップロード (POST /contracts/{id}/documents)
  → 契約情報の保存 (PUT /contracts/{id}/fields)
  → 相手方・承認者の登録 (POST /contracts/{id}/partner_companies)
  → 送信 (POST /contracts/{id}/confirm)   ← ここでMFから署名依頼メールが飛ぶ
  → [発注者がMFの画面で署名]
  → 合意締結証明書の取得 (GET /contracts/{id}/certificate)
```

`remind` があるため署名の催促を自動化でき、決済のオーソリ期限（約7日）対策として有効。
`withdraw` があるため締結不成立時に取下げ→決済キャンセルを連動できる。

#### ⚠ 残る制約（設計に影響する）

| # | 制約 | 影響と対応 |
| --- | --- | --- |
| 1 | **Webhook がない** | 締結完了はポーリングで検知する。Cloud Scheduler + Cloud Tasks で定期実行する |
| 2 | **締結状況を取得するエンドポイントがない** | `presenter.Contract` に status フィールドがなく、`GET /contracts` `GET /contracts/{id}` はいずれも**下書きステータス専用**と明記されている。締結完了の判定は **`GET /contracts/{id}/certificate` が成功するかどうか**で行うのが現実的。Phase 0 で実挙動（未締結時のステータスコード）を検証する |
| 3 | **締結済PDF本体をダウンロードするエンドポイントがない** | **永続保管をMFに委ねる方針（[adr/0004](adr/0004-delegate-retention-to-moneyforward.md)）により、この制約の影響は小さくなった。** 自システムは参照IDとメタデータだけを持ち、実体はMF側に置く。締結完了の検知には合意締結証明書の取得可否を使う（制約2） |
| 4 | **関連書類の添付専用のエンドポイントがない** | `POST /contracts/{id}/documents` は「契約書PDFファイルを追加」。受領図面を別ファイルとして添付できるかは未検証。**受領図面は契約書PDFに結合して1つのPDFにする**方針を既定とする。署名対象そのものに図面が含まれるため、「契約時の条件が曖昧にならないように」という要件にはむしろ適合する |
| 5 | **署名画面の制御はできない** | 従来どおり。決済は自サイトで完結させる（下記） |
| 6 | 認証が `x-email` に紐づく | 個人ユーザーに紐づく場合、退職・異動でAPIが止まる。**API専用ユーザーを作成して運用する**。認証情報は Secret Manager に置く |
| 7 | エラー形式が仕様と実挙動で異なる | 仕様の `presenter.Error` は `{code, detail, source, status}` だが、実レスポンスは `{"errors":[{type, code, message, param}]}`。**エラーハンドリングは実レスポンスに合わせる**（仕様を信じない） |

#### 参照した仕様の取得方法

仕様は Swagger UI から `doc.json` として取得できる。
バージョン差分を確認したいときは以下で取得する（リポジトリには含めない）。

```sh
curl -s https://api.contract.moneyforward.com/v1/docs/doc.json -o mf-contract-v1.json
```

### 署名画面と決済の同居について（要件への回答）

**「MFのメールリンク先の画面で決済してもらう」ことはできない。**
署名画面はMFのドメインで提供され、UIの差し込みも遷移先の制御も第三者にはできない。
API にも署名完了後のリダイレクトURLを指定するパラメータは存在しない。

したがって **決済は自サイト内で完結させ、決済（与信確保）を署名依頼送信（`confirm`）のトリガーとする**。
→ [02-business-flow.md §2.5](02-business-flow.md)

### 保管の委譲

締結済契約書・合意締結証明書の**永続保管はMFクラウド契約に委ねる**（→ [adr/0004](adr/0004-delegate-retention-to-moneyforward.md)）。
一次ソースで確認した根拠:

- 送信料・保管料0円、件数による課金・上限なし（[料金ページ](https://biz.moneyforward.com/contract/price/)）
- すべての電子契約に10年間の長期署名（[機能ページ](https://biz.moneyforward.com/contract/function/)）
- 電子帳簿保存法の「電子取引」に対応。**ただし「契約書名・契約開始日・終了日・契約金額」を
  すべて入力することが条件**（[MF公式サポート](https://biz.moneyforward.com/support/contract/guide/service-guide/g020.html)）。
  「スキャナ保存」には非対応
- 解約すると**無償利用状態に移行し閲覧・出力ができなくなる場合がある**。退会後はダウンロード不可
  （[解約FAQ](https://biz.moneyforward.com/support/plan/faq/te07.html)）

→ `PUT /contracts/{id}/fields` は必須処理。解約時のエクスポート手段は Phase 0-10 で確認する。

### プロバイダ抽象化は維持する

API が使えると判明したため、`MoneyForwardApiProvider` が本命の実装になる。
それでも `EsignProvider` による抽象化は維持する（→ [adr/0002](adr/0002-esign-provider-abstraction.md)）。理由:

- Webhook がなく、締結状況の取得手段も間接的であるため、**この部分の実装は今後変わる可能性が高い**
- 開発者サイトに掲載されていないAPIであり、変更ポリシーやサポート範囲が不明
- テスト用のフェイク実装が必要

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

| # | 事項 | 影響範囲 | 確認先 |
| --- | --- | --- | --- |
| 1 | **クラウド契約 API の利用条件**: 必要プラン、`x-token` の発行方法、利用規約、サポート範囲、レート制限、変更ポリシー | 全体設計。API利用可否が決まる | MF営業・サポート |
| 2 | **未締結時に `GET /contracts/{id}/certificate` が何を返すか** | 締結完了の判定ロジック | 実挙動の検証 |
| 3 | **`POST /contracts/{id}/documents` に複数ファイルを追加できるか** | 受領図面を別ファイル添付にするか、契約書PDFへ結合するか | 実挙動の検証 |
| 4 | **締結済PDF本体の取得手段**（APIにエンドポイントがない） | 自システムでの保管方法 | MF・実挙動の検証 |
| 5 | `x-email` が個人ユーザーに紐づくか（退職・異動時の失効リスク） | API専用ユーザーの要否 | MF |
| 6 | 署名依頼メールの文面カスタマイズ可否 | 発注者体験 | MF |
| 7 | 現在の契約プランで送信・締結機能が使えるか | 即時 | MF |
| 8 | 電子署名法上の署名方式（事業者署名型／当事者署名型） | 法的効力の説明資料 | MF |
| 9 | IT重説（ビデオ通話での免許証提示）の適法性 | 重説フローの実装 | 顧問弁護士／建築士事務所協会 |
| 10 | 電磁的方法の承諾を案件ごとに取るか包括で取るか | 承諾画面の設計 | 顧問弁護士 |
| 11 | 消費者契約における特商法の適用区分（通信販売／訪問販売） | クーリング・オフ、表示義務 | 顧問弁護士 |
| 12 | 改正法の**施行期日政令**（実際の施行日）と、施行に伴う政省令改正の内容 | スケジュールと法定必須項目 | 国土交通省の公布を待つ |
| 13 | **解約・退会時の一括エクスポート手段** | 永続保管をMFに委ねる前提が崩れないか | MF |
| 14 | `fields` の4項目を入れた場合の電帳法要件の充足（実挙動） | 保存要件 | 実挙動の検証 |

**解消済み**: 「見積書の規定が努力義務か義務か」は、改正法本文（令和8年法律第74号 第24条の10）で
**努力義務**であることを確認した。監督処分の対象にも含まれない（法26条2項1号の列挙外）。
→ [01-legal-requirements.md §1.5](01-legal-requirements.md)

> 2・3・4 は API の利用が可能になった時点でサンドボックスで検証する（Phase 0-1b）。
> 仕様書の記述だけでは判断できないため、必ず実挙動を確認すること。
