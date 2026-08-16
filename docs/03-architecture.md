# アーキテクチャ

## 1. 構成

```
                    ┌─────────────────────────────────────┐
                    │ Firebase Hosting (静的アセット/SPA)   │
                    │  - 事務所向け管理画面                 │
                    │  - 発注者向け契約画面                 │
                    └──────────────┬──────────────────────┘
                                   │ /api/** を rewrite
                                   ↓
                    ┌─────────────────────────────────────┐
                    │ Cloud Run (API: TypeScript/Hono)     │
                    └──┬────────┬────────┬────────┬───────┘
                       │        │        │        │
        ┌──────────────┘        │        │        └──────────────┐
        ↓                       ↓        ↓                       ↓
┌───────────────┐  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐
│ Cloud SQL      │  │ Cloud Storage  │  │ Secret       │  │ 外部サービス   │
│ (PostgreSQL)   │  │ (一時領域/TTL)  │  │ Manager      │  │ MF / Stripe  │
└───────────────┘  └────────────────┘  └──────────────┘  │ Clerk / Mail │
                                                          └──────────────┘
        ┌─────────────────────────────────────┐
        │ Cloud Tasks / Cloud Scheduler        │
        │  - オーソリ期限通知                    │
        │  - 契約期限アラート                    │
        └─────────────────────────────────────┘
```

- **締結の状態は本システムが持つ。** 電子署名を用いず交付を自前で完結させるため
  （→ [adr/0005](adr/0005-electronic-signature-requirement.md)）、外部サービスへの
  ポーリングも Webhook 受信も不要になった。締結完了は発注者の受領確認をもって確定する
  （→ [02-business-flow.md §2.6](02-business-flow.md)）。
  Webhook を受けるのは **Stripe だけ**（署名検証必須・冪等処理）。

- **Firebase Hosting の rewrite で Cloud Run にプロキシ**し、単一オリジンで運用する。
  クロスオリジンのCookie問題とCORS設定を避けられる。
- Cloud Run は最小インスタンス0で開始してよい。ただし発注者向け画面のコールドスタートが
  体感を損なうため、本番稼働後は `min-instances=1` を検討する。
- リージョンは `asia-northeast1`（東京）。契約データの所在地を国内に固定する。
- **本システムは永続保管しない。** Cloud Storage は作業領域と引き渡しの猶予に限り、
  確定書面は事務所の手元（Drive 等）へ引き渡す（→ §4、[adr/0006](adr/0006-self-hosted-retention.md)）。

## 2. 認証・認可

| 利用者 | 認証 | 備考 |
| --- | --- | --- |
| 事務所スタッフ（管理建築士・所員） | **Clerk** | メール＋パスワード＋MFA必須。ロール: `admin` / `architect` / `staff` |
| 発注者（建築主） | **署名付きリンク（トークン）** | Clerkのアカウント作成を求めない |

**発注者にアカウントを作らせない**のは意図的な設計判断である。

- 一般消費者が住宅設計を1回発注するのに、パスワード管理を強いるのは離脱要因になる。
- Clerk の MAU 課金対象を、実質1回しか使わない発注者で膨らませない。
- 代わりにトークンの安全性を担保する:
  - 32バイトの乱数を base64url で発行。DBには SHA-256 ハッシュのみ保存。
  - 有効期限（既定30日）、失効操作、利用回数の記録。
  - 初回アクセス時にメールアドレスの照合を行う（送信先アドレスの入力を求める）。
  - 決済・署名依頼など重要操作の直前にワンタイムコードをメール送信して再確認する
    （ステップアップ認証）。
- トークンは `Referer` 経由で外部に漏れうるため、**URLはパスに置きトークンはクエリに置かない**、
  かつ `Referrer-Policy: same-origin` を設定する。

`architect` ロールのユーザーには**建築士登録番号・資格区分・管理建築士フラグ**を属性として持たせ、
重要事項説明の実施記録・契約書の法定記載事項（従事建築士）に流用する。

## 3. データモデル（概要）

契約種別ごとの差異を型で表現し、汎用性を持たせる。

```
Client（契約の相手方）
 ├─ id, kind(individual|corporation), name, kana, address, email, tel
 ├─ is_consumer（消費者契約法・特商法の適用判定に使う）
 └─ is_building_owner（建築主か。法24条の7の適用判定に使う。
     元請け設計事務所からの受託など、相手方が建築主でない場合は false）

Project（案件）
 ├─ id, client_id, name
 ├─ building（建物の概要：用途、構造、規模、階数、延べ面積、所在地、工事種別）
 │    ※ 延べ面積は法24条の3第2項（一括再委託の禁止／300㎡超の新築工事）の判定に使う。
 │      法22条の3の3の適用判定には使わない（改正で面積要件が撤廃されるため）
 └─ documents[]（受領資料）

Document（資料・成果物）
 ├─ id, project_id, kind(received_drawing|quote|important_matters|contract)
 ├─ storage_uri, filename, content_type, byte_size, page_count
 ├─ sha256, hash_algorithm（既定 'sha256'）
 ├─ finalized_at（**確定日時。立った時点でバイト列が固定され、再生成を禁止する**）
 ├─ prefix(draft|executed)（ライフサイクルが異なる。→ §4）
 ├─ expires_at（TTLによる削除予定日。draft 90日／executed 365日）
 ├─ archived_at（**事務所が手元へ保存した記録。null なら未保管**）
 └─ received_at, uploaded_by

 ※ **実体はいずれTTLで消える。** 確定PDFの永続保管は事務所の手元で行う（→ adr/0006）。
   archived_at が null のまま expires_at に近づいたらアラートする（→ §4）。
   sha256 は実体が消えた後も残す。**相手方によるハッシュ照合は実体なしでも成立する。**
   確定PDFの再生成はハッシュ不一致を招くため、コードレベルで禁止する（→ adr/0005）。

DocumentDelivery（交付の記録）※ 規則17条の39・22条の2の3 の充足を示す証跡
 ├─ id, document_id, recipient（client_token_id または email）
 ├─ method(electronic_organization|browsing|paper)
 ├─ delivered_at
 ├─ hash_notified_at, notification_message_id（通知メールのMessage-ID。送達記録）
 ├─ read_confirmed_at（閲覧確認。規則17条の39第2項3号ただし書の証跡）
 ├─ received_at（相手方の受領確認。相互交付の成立判定に使う）
 └─ integrity_verified_at（相手方が検証ページでハッシュを照合した記録。任意）

ContractType（契約種別マスタ）
 ├─ code: design | construction_supervision | seismic_diagnosis
 │        | seismic_retrofit_design | inspection | procedure_agency | other
 ├─ requires_mutual_delivery（法22条の3の3の適用有無。義務 → 締結をブロックする）
 ├─ may_require_important_matters（法24条の7の適用「候補」。
 │    実際の要否は Client.is_building_owner と合わせて判定する。マスタだけでは決まらない）
 ├─ recommends_quote（法24条の10の対象か。努力義務 → 警告のみ。ブロックしない）
 ├─ fee_standard: kokuji8 | kokuji670 | none（適用する業務報酬基準）
 ├─ template_id
 └─ input_schema（Zodスキーマ識別子）

Contract（契約）
 ├─ id, project_id, contract_type_code, status, version, parent_contract_id
 ├─ terms: jsonb（法定記載事項を含む契約条件のスナップショット）
 ├─ fee_amount, fee_tax, payment_schedule[]
 ├─ attached_document_ids[]（別紙目録）
 ├─ esign_provider（self_hosted | moneyforward | manual）
 ├─ esign_document_id（外部サービスを使う場合の参照。self_hosted では null）
 ├─ contract_document_id（**確定した契約書PDF。self_hosted ではこれが正本**）
 ├─ executed_at（相互交付の成立日時。→ 02-business-flow.md §2.6）
 └─ created_at, updated_at

ElectronicDeliveryConsent（電磁的方法の承諾／令8条・令7条）
 ├─ id, contract_id, disclosure_text_version, disclosed_text_snapshot
 ├─ consented_at, consented_by_name, ip, user_agent
 └─ withdrawn_at, withdrawal_reason

ImportantMattersExplanation（重要事項説明の実施記録／法24条の7）
 ├─ id, contract_id, document_id（交付した重説書面）
 ├─ explained_at, method(in_person|video), explainer_user_id
 ├─ explainer_name, explainer_license_class, explainer_license_number
 ├─ license_presented（免許証提示の方法）
 ├─ acknowledged_at（建築主の確認）
 │
 │  ※ method='video'（IT重説）の場合、国交省の実施マニュアルが定める
 │    遵守6項目の記録が必要（→ 01-legal-requirements.md §3.3）
 ├─ it_consent_at（① 建築主の意向確認・事前同意）
 ├─ it_environment_checked（② IT環境の確認内容）
 ├─ it_precheck（④ 映像・音声・書面確認の3点チェック）
 ├─ owner_id_verification（⑤ 建築主の本人確認。提示された書類の種類）
 └─ license_visual_confirmation（⑥ 免許証の視認確認方法。
      氏名の読み上げ／顔写真との照合など、**何をどう確認したか**を残す）

Payment（決済）
 ├─ id, contract_id, provider(stripe), payment_intent_id
 ├─ kind(deposit|advance|full), amount, currency
 └─ authorized_at, captured_at, canceled_at, refunded_at

ContractEvent（監査証跡・追記専用）
 ├─ id, contract_id, type, actor(staff:{user_id}|client:{token_id}|system)
 ├─ payload jsonb, occurred_at
 ├─ ※ UPDATE / DELETE を DBロールレベルで禁止する
 └─ type の例:
      consent.granted / consent.withdrawn        電磁的方法の承諾・撤回
      document.finalized                         PDF確定（sha256 を payload に含める）
      document.delivered                         交付
      document.hash_notified                     ハッシュ値の通知（message_id を含める）
      document.read_confirmed                    閲覧確認
      document.received                          相手方の受領確認
      document.integrity_verified                相手方によるハッシュ照合
      document.archived                          事務所が手元へ保存（→ adr/0006）
      contract.executed / contract.withdrawn     締結・取下げ
```

> **`ContractEvent` は監査証跡であって、電帳法の要件充足の根拠ではない。**
> 規則4条1項3号イは**保存もそのシステムで行うこと**を求めており、
> 保管を事務所の手元に置く以上この号は使えない（→ §6、[adr/0006](adr/0006-self-hosted-retention.md)）。
> それでも紛争時の裏付けとしては中核なので、UPDATE/DELETE の禁止は
> アプリケーション層ではなく**DBロールレベル**で強制する。

### 契約条件のスナップショット

**電子署名を用いず、実体も手元へ移るため、この項目の重要度はきわめて高い。**
GCS上のファイルはTTLで消えるが、`Document.sha256`・`Contract.terms`・`DocumentDelivery`・
`contract_events` が揃っていれば、**「何をどういう条件で締結し、いつ誰に渡したか」を
自システムだけで説明できる。** ここが薄いと、実体を失った時点で何も示せなくなる。

`Contract.terms` には契約時点の法定記載事項を**値としてコピーして固定**する。
建築士事務所名や従事建築士をマスタから参照するだけにすると、
マスタ更新で過去の契約書の再生成結果が変わってしまう。締結済み契約書の再現性を守るため、
締結時点の値をスナップショットする。

### 法定必須項目のバリデーション

`requires_mutual_delivery = true` の契約種別では、
[01-legal-requirements.md §2](01-legal-requirements.md) の項目を**サーバ側で必須検証**する。
重説書面は必須項目セットが異なる（規則17条の38の1号〜6号のみ）ため、
`contractFormSchema` と `importantMattersSchema` を**別のZodスキーマとして定義**し、
共通部分のみ合成する。

**3つの義務は強制の強さも適用条件も違う。混ぜないこと**（→ [01-legal-requirements.md §5](01-legal-requirements.md)）。

| 義務 | 根拠 | 適用条件 | 違反時の扱い |
| --- | --- | --- | --- |
| 書面の相互交付 | 法22条の3の3（義務・監督処分あり） | `ContractType.requires_mutual_delivery` | **409 を返して締結を止める** |
| 重要事項説明 | 法24条の7（義務・監督処分あり） | `may_require_important_matters` **かつ** `Client.is_building_owner` | **409 を返して締結を止める** |
| 見積書 | 法24条の10（努力義務・監督処分なし） | `ContractType.recommends_quote` | **止めない。** 警告を返して続行できる |

**重要事項説明の要否は契約種別だけでは決まらない。** 法24条の7第1項は
「設計受託契約又は工事監理受託契約を**建築主と**締結しようとするとき」と定めており、
元請け設計事務所からの受託など**相手方が建築主でない場合には適用がない**。
一方、書面の相互交付（22条の3の3）は「当事者は」であり建築主に限られないため、
この2つを同じフラグで扱うと誤る。

```ts
/** 重要事項説明の要否（法24条の7第1項: 「建築主と締結しようとするとき」） */
export function requiresImportantMattersExplanation(
  type: ContractType,
  counterparty: { isBuildingOwner: boolean },
): boolean {
  return type.mayRequireImportantMatters && counterparty.isBuildingOwner
}
```

建築主でない相手にも重説相当の説明を任意で行うのは差し支えないが、
**締結のブロック条件にしてはならない**（義務がない以上、止める根拠がない）。

3つの対象範囲は一致しない。とくに `recommends_quote` は
書面契約義務が及ばない調査・耐震診断にも立つ。「だいたい同じだから」で1つに畳まない。

### 報酬算定は業務報酬基準の告示ごとに分ける

`fee_standard` で選択する。告示第8号と告示第670号は**費目の構成が違う**
（第670号には検査費がある）ため、1つの計算器に押し込めない。
→ [01-legal-requirements.md §1.7](01-legal-requirements.md)

### 再委託（法24条の3）

再委託先を契約書に記載する場合（規則17条の38第6号）、次を持たせる。

```
Subcontract（再委託）
 ├─ id, contract_id, scope（委託する業務の概要）
 ├─ contractor_name, office_name, office_address
 ├─ is_registered_office（相手が建築士事務所の開設者か。**法24条の3第1項により必須**）
 └─ acknowledged_not_wholesale（300㎡超の新築工事で一括再委託でないことの確認。法24条の3第2項）
```

- 法24条の3第1項は**委託者の許諾があっても**建築士事務所の開設者以外への再委託を禁じる。
  `is_registered_office = false` は登録できないようにする。
- 同2項の一括再委託の禁止は**延べ面積300㎡超の新築工事に限る**。
  「一括」かどうかは業務範囲の実質で決まり自動判定できないため、該当条件のときに確認項目を出す
  （ブロックはしない）。
- → [01-legal-requirements.md §6](01-legal-requirements.md)

### 適用法令バージョン

改正法附則第4条により、法22条の3の3は**施行日以後に締結される契約**に適用される。
`Contract` は締結日を保持し、そこから適用法令バージョン（改正前／改正後）を判定する。
施行日をまたぐ案件があるため、この判定をハードコードした定数で分岐させない。

## 4. ストレージ方針

**本システムは永続保管しない。確定した書面を確実に事務所の手元へ引き渡すことに責任を持つ。**
→ 決定の経緯は [adr/0006-self-hosted-retention.md](adr/0006-self-hosted-retention.md)
（[adr/0004](adr/0004-delegate-retention-to-moneyforward.md) を supersede）

保管先は事務所が管理する場所（Google Drive 等）とし、システムは関与しない。

| データ | 正（永続保管） | 本システムでの扱い |
| --- | --- | --- |
| **確定した契約書PDF** | **事務所の手元**（Drive 等） | `-workspace` の `executed/` に**TTL 365日**で置き、メールとダウンロードで引き渡す |
| **重要事項説明書PDF** | 同上 | 同上 |
| 見積書 | MFクラウド請求書（API化は Phase 4） | 参照ID（`quote_id`）と金額・日付をDBに保持 |
| 受領図面・確定前のドラフト | — | `-workspace` に TTL 90日。契約に至らなければ消える |
| **契約条件のスナップショット・ハッシュ値・承諾記録・交付記録・重説実施記録・監査証跡** | **Cloud SQL** | **これは自システムが正。実体が手元に移っても残す** |

> **`executed/` の TTL 365日は「保管」ではなく「引き渡しの猶予」。**
> ハッシュ値（`Document.sha256`）は Cloud SQL に残るため、**実体がGCSから消えても
> 相手方によるハッシュ照合は成立する**（相手方が自分の手元のPDFを計算し、通知メールの値と比べる）。

### なぜ Google Drive は使わないのか

一時領域であっても Drive は使わない。理由は [adr/0003](adr/0003-object-storage-gcs-over-google-drive.md) のとおりで、
永続・一時の別によらず次の2点が効く。

- 「リンクを知っている全員」への共有が**UIから1クリックで**できてしまう（契約書・図面では致命的）
- アカウントを持たない発注者への**期限付き公開（署名付きURL）**が Drive では作れない

Drive は「人が図面をやりとり・共同編集する作業場所」として従来どおり残し、
そこから本システムへ取り込む導線（Drive Picker またはアップロード）を設ける。
取り込み時に GCS へコピーして SHA-256 を固定する。

### GCS のバケット設計

**バケットは1つ**（`-workspace`）。永続保管をしないため、用途別に分ける必要がない。
プレフィックスでライフサイクルを分ける。

| 設定 | 値 | 理由 |
| --- | --- | --- |
| ライフサイクル（既定） | **作成から90日で自動削除** | 作業領域であることを構成で強制する |
| ライフサイクル（`executed/`） | **作成から365日で自動削除** | **引き渡しの猶予。** 90日では短すぎて事故を招く |
| uniform bucket-level access | 有効 | ACLの個別付与を封じる |
| 公開アクセス防止 | 組織ポリシーで強制 | 誤公開の経路を消す |
| CMEK | 適用 | 図面・個人情報を含む |
| バージョニング | 不要 | 永続保管しないため |
| バケットロック（保持ポリシー） | **不要** | 保管義務はシステムでは担保しない（→ [adr/0006](adr/0006-self-hosted-retention.md)） |

発注者へのファイル配布は署名付きURL（有効期限15分）。

**既定の TTL 90日は「契約成立までの猶予」**である。見積の有効期限（既定30日）＋確認待ち＋余裕を見た。
**`executed/` の TTL 365日は「事務所が手元へ保存するまでの猶予」**であり、保管ではない。
滞留量は最大1年分に収まる。

#### 確定の手順（ここが法令要件と直結する）

```
ドラフト生成（-workspace）
  → 内容確定
  → PDF/A-2b で出力し、そのバイト列を -workspace の executed/ に格納
  → SHA-256 を計算して Document.sha256 に固定、finalized_at を立てる
  → 以後、再生成しない
  → 事務所宛にPDF添付でメール送信、発注者宛に交付通知
  → 事務所が Drive 等へ保存し、archived_at を記録
```

**確定後のPDFを再生成してはならない。** PDF生成は生成時刻の埋め込みやフォントのサブセット化により
非決定的になりうるため、再生成するとハッシュが一致しなくなる（→ [adr/0005](adr/0005-electronic-signature-requirement.md)）。
訂正が必要な場合は新しい版を作り、`Contract.version` と `parent_contract_id` で履歴を残す。

#### 未保管の検知（運用依存を埋める中核）

**保存義務の履行が構成ではなく運用に依存するため、取りこぼしを検知する仕組みを Phase 1 に含める。**

- 管理画面に**未保管一覧**（`finalized_at` が立っていて `archived_at` が null）を常設する
- **TTL 期限が近い未保管の書面をアラートする**（Cloud Scheduler で日次チェック）
- 確定と同時に事務所宛メールへ**PDFを添付**する（気づかせる第一の手段）

> **消えたら取り返せない。** ここを後回しにすると、電帳法の保存義務違反が静かに発生する。

## 5. セキュリティ

- **秘匿情報**は Secret Manager。環境変数への直書きを禁止（→ CODING_STANDARDS）。
- **Webhook 署名検証**: Stripe は署名検証必須。受信する Webhook は Stripe だけ（→ §1）。
- **PII のログ出力禁止**: 氏名・住所・メールアドレス・電話番号・トークンをログに出さない。
  ロガーでフィールドをマスクする仕組みを共通化する（→ CODING_STANDARDS §7）。
- **CSP**: 契約書PDFの表示にインラインビューアを使う場合、`frame-ancestors 'none'` と
  厳格な `script-src` を設定する。
- **レート制限**: 発注者向けトークンエンドポイントにIP・トークン単位のレート制限を入れる
  （トークン総当たり対策）。
- **バックアップ**: Cloud SQL の自動バックアップ＋ PITR。復旧手順を年1回演習する。
- **委託先管理**: MF・Stripe・Clerk・Google Cloud を個人情報の委託先として一覧化し、
  安全管理措置の確認記録を残す（個人情報保護法）。

## 6. 電子帳簿保存法への対応

**保存は事務所の手元で行う**（→ [adr/0006](adr/0006-self-hosted-retention.md)）。
**保存場所がシステム外に出るため、要件の満たし方が変わる。**

### 真実性の確保 — 4号（事務処理規程）が唯一の手段になる

電子取引の取引情報に係る電磁的記録は、次の**いずれか**の措置を行って保存する
（電子帳簿保存法施行規則4条1項）。

| # | 措置 | 採否 |
| --- | --- | --- |
| 1 | タイムスタンプが付された後に授受 | — |
| 2 | 授受後速やかにタイムスタンプを付す | — |
| 3イ | 訂正又は削除を行った場合に、これらの事実及び内容を確認することができるシステムで**授受及び保存**を行う | **使えない**（下記） |
| 3ロ | 訂正又は削除を行うことができないシステム | 同上 |
| **4** | 正当な理由がない訂正及び削除の防止に関する**事務処理の規程**を定め、規程に沿った運用を行い、規程の備付けを行う | **採用。唯一の手段** |

> **なぜ3号イが使えないのか。** 条文は「当該取引情報の**授受及び当該電磁的記録の保存**を行うこと」と
> 定めており、**保存もそのシステムで行うことを求めている**。
> Drive 等での保存はこれに当たらない。`ContractEvent` が追記専用であることは
> 監査証跡としては有効だが、**電帳法の要件充足の根拠にはならない。**

> **タイムスタンプも採らない。** 電帳法上の「タイムスタンプ」は
> **総務大臣が認定する時刻認証業務に係るもの**に限定されており（規則2条6項2号ロ）、有償サービスが必要。

**4号の事務処理規程は Phase 0-10 で整備する。規程を作っただけでは足りず、
運用と備付けまで揃って要件充足になる。**

> ⚠ **これはリリースのブロッカーである。** 代替手段（3号イ）がなくなったため、
> 規程が未整備のまま運用を始めると、**真実性の要件を満たす手段がゼロになる。**

### 可視性の確保 — 小規模事業者の特例で検索要件が外れる見込み

規則4条1項の柱書は、次の場合に検索要件を**除く**と定める。

> 当該保存義務者が、その**判定期間に係る基準期間における売上高が五千万円以下である事業者である場合**
> （中略）**であって、当該電磁的記録の提示等の要求に応じることができるようにしているとき**は、
> 同号に掲げる要件（＝検索機能）を除く。

**1名の建築士事務所は該当する見込みが高い。** ダウンロードの求めに応じられるようにしておけば、
検索機能の実装は不要になる。

該当しなくなる可能性に備え、**ファイル名規則を運用に入れる**。

```
YYYYMMDD_取引先名_金額.pdf     例: 20270415_山田太郎_3300000.pdf
```

システムが生成するPDFのファイル名をこの規則に従わせ、
**そのまま Drive へ保存すれば検索要件を満たせる**状態にする。

ディスプレイ・プリンタ等の備付けと、整然・明瞭な出力ができることは、
Drive のプレビューと印刷で満たす。

### 残る課題（Phase 0 で詰める）

| # | 課題 | 内容 |
| --- | --- | --- |
| 1 | ~~解約・退会時のデータ~~ | **消滅。** 外部サービスに保管を委ねないため |
| 2 | **紙面フォールバック分** | 相手方が電磁的方法を承諾しない場合の紙契約。**スキャナ保存**の要件（解像度200dpi以上・256階調以上・認定タイムスタンプ）を満たすのは負担が大きい。**紙は紙で保存する**を既定とし、電子化は将来の検討事項とする |
| 3 | ~~重要事項説明書の保管~~ | **決定済み。** 契約書と同じく事務所の手元で保管する（交付義務の履行を示す証跡。法24条の7・監督処分の対象） |
| 4 | **失注案件の受領図面** | 契約に至らなければ `-workspace` のTTLで消える。見積のやり直しや紛争時に参照したいケースがないかを運用で確認する |
| 5 | **事務処理規程の整備** | 規則4条1項4号。**リリースのブロッカー。** 規程の策定・運用・備付け |
| 6 | **保管先の共有設定ルール**（新規） | Drive 等に契約書・図面を置くため、フォルダの共有設定を誤ると個人情報が漏えいする。Phase 0-7（個人情報取扱規程）に含める |

