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
        │  - 締結状況ポーリング                  │
        │  - オーソリ期限通知                    │
        │  - 契約期限アラート                    │
        └─────────────────────────────────────┘
```

- **Firebase Hosting の rewrite で Cloud Run にプロキシ**し、単一オリジンで運用する。
  クロスオリジンのCookie問題とCORS設定を避けられる。
- Cloud Run は最小インスタンス0で開始してよい。ただし発注者向け画面のコールドスタートが
  体感を損なうため、本番稼働後は `min-instances=1` を検討する。
- リージョンは `asia-northeast1`（東京）。契約データの所在地を国内に固定する。
- **締結済契約書の永続保管はマネーフォワード クラウド契約側**。Cloud Storage は契約書を作るための
  一時領域に限る（→ §4、[adr/0004](adr/0004-delegate-retention-to-moneyforward.md)）。

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

Document（資料・成果物）※ 実体は一時領域。TTLで消えることを前提に扱う
 ├─ id, project_id, kind(received_drawing|quote|important_matters|contract)
 ├─ storage_uri（一時。期限切れ後は null）, filename, content_type, byte_size, page_count
 ├─ sha256（受領時ハッシュ。**実体が消えてもレコードは残す**。別紙目録の再現に使う）
 ├─ expires_at（ライフサイクルによる削除予定日）
 └─ received_at, uploaded_by

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
 ├─ esign_provider, esign_document_id（**MF側の正本への参照。実体は持たない**）
 ├─ executed_at
 ├─ mf_fields_saved_at（電帳法要件の4項目をMFへ保存した日時。→ §6）
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
 └─ acknowledged_at（建築主の確認）

Payment（決済）
 ├─ id, contract_id, provider(stripe), payment_intent_id
 ├─ kind(deposit|advance|full), amount, currency
 └─ authorized_at, captured_at, canceled_at, refunded_at

ContractEvent（監査証跡・追記専用）
 ├─ id, contract_id, type, actor(staff:{user_id}|client:{token_id}|system)
 ├─ payload jsonb, occurred_at
 └─ ※ UPDATE / DELETE を DBロールレベルで禁止する
```

### 契約条件のスナップショット

**永続保管をMFに委ねるため、この項目の重要度が上がっている。**
ファイルの実体は手元から消えるが、`Contract.terms` と `contract_events` が残っていれば
「何をどういう条件で締結したか」は自システムだけで説明できる。

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

**永続保管はマネーフォワード クラウド契約に委ねる。Cloud Storage は契約書を作るための一時領域に限る。**
→ 決定の経緯は [adr/0004-delegate-retention-to-moneyforward.md](adr/0004-delegate-retention-to-moneyforward.md)

| データ | 正（永続保管） | 本システムでの扱い |
| --- | --- | --- |
| 締結済契約書・合意締結証明書 | **MFクラウド契約** | 参照ID（`esign_document_id`）とメタデータのみDBに保持。実体は持たない |
| 見積書 | **MFクラウド請求書** | 同上（`quote_id` と金額・日付） |
| 受領図面・生成した契約書ドラフト・重説書面 | **一時領域（GCS）** | 契約書PDFに結合してMFへ送るまでの作業用。TTLで自動削除 |
| 契約条件のスナップショット・承諾記録・重説実施記録・監査証跡 | **Cloud SQL** | これは自システムが正。ファイルではなく構造化データ |

### なぜ Google Drive は使わないのか

一時領域であっても Drive は使わない。理由は [adr/0003](adr/0003-object-storage-gcs-over-google-drive.md) のとおりで、
永続・一時の別によらず次の2点が効く。

- 「リンクを知っている全員」への共有が**UIから1クリックで**できてしまう（契約書・図面では致命的）
- アカウントを持たない発注者への**期限付き公開（署名付きURL）**が Drive では作れない

Drive は「人が図面をやりとり・共同編集する作業場所」として従来どおり残し、
そこから本システムへ取り込む導線（Drive Picker またはアップロード）を設ける。
取り込み時に GCS へコピーして SHA-256 を固定する。

### GCS のバケット設計

**バケットは1つ**（`-workspace`）。永続保管をしないため、用途別に分ける必要がなくなった。

| 設定 | 値 | 理由 |
| --- | --- | --- |
| ライフサイクル | **オブジェクト作成から90日で自動削除** | 一時領域であることを構成で強制する。運用の判断に委ねない |
| uniform bucket-level access | 有効 | ACLの個別付与を封じる |
| 公開アクセス防止 | 組織ポリシーで強制 | 誤公開の経路を消す |
| CMEK | 適用 | 一時とはいえ図面・個人情報を含む |
| バージョニング | **不要** | 永続保管しないため。誤削除は再生成できる |
| バケットロック（保持ポリシー） | **不要** | 保管義務を負わないため |

発注者へのファイル配布は署名付きURL（有効期限15分）。

**TTL は「契約成立までの猶予」を決める設定値**である。見積の有効期限（既定30日）＋署名待ち＋余裕を見て90日とした。
これを超えて残す必要がある案件が出たら、TTLを延ばすのではなく**MFへ送る**か、意識的に手元へ退避する。

## 5. セキュリティ

- **秘匿情報**は Secret Manager。環境変数への直書きを禁止（→ CODING_STANDARDS）。
- **Webhook 署名検証**: Stripe は署名検証必須。MFがWebhookを提供する場合も同様。
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

**保存要件はマネーフォワード クラウド契約側で満たす。** 本システムは要件を満たすための入力を
確実に行うことに責任を持つ。

### MF側の対応範囲（一次ソース: MF公式サポート）

- クラウド契約は電子帳簿保存法における**「電子取引」に対応**している。
  締結された電子契約にはタイムスタンプが付与される。
- **「スキャナ保存」には対応していない。**
- **条件付きである点に注意**: MFの案内は
  「**『契約書名』『契約開始日・終了日』『契約金額』の情報をすべて入力すると、
  『電子取引』の要件を満たすことが可能**」と書かれている。
  → [電子帳簿保存法への対応について（MFクラウド契約サポート）](https://biz.moneyforward.com/support/contract/guide/service-guide/g020.html)

### 実装への影響（重要）

**`PUT /contracts/{id}/fields` は「検索性のための付随的な処理」ではなく、電帳法要件を満たすための必須処理である。**
契約書名・契約開始日・終了日・契約金額を**必ず**送る。送信前にこの4項目が揃っていることを検証し、
欠けていたら締結依頼（`confirm`）に進ませない。

> 以前は fields を付随的な処理と位置付けていたが、永続保管をMFに委ねる以上、
> ここが欠けると保存要件を満たさない。位置付けを格上げした。
> なお**法定記載事項の記載場所は契約書PDF本体**であることは変わらない（fields はメタデータ）。
> 建築士法の要件と電帳法の要件は別物なので混同しないこと。

### 自システムが持つもの

ファイル実体を持たない代わりに、**検索と追跡ができる最小限のメタデータ**を Cloud SQL に持つ。

- `contracts`: 取引年月日（`executed_at`）・取引金額（`fee_amount`）・取引先（`client_id`）と
  MF側の参照ID（`esign_document_id`）
- `contract_events`: 監査証跡（追記専用）

税務調査等でダウンロードを求められた場合は、**MFの画面から出力する**。
自システムは「どの契約がMFのどのIDに対応するか」を引ける状態を保つ。

### 残る課題（Phase 0 で詰める）

| # | 課題 | 内容 |
| --- | --- | --- |
| 1 | **解約・退会時のデータ** | MF公式FAQによれば、有償契約を解約しても登録データは削除されないが**無償利用状態に移行し、閲覧・出力ができなくなる場合がある**。退会（事業者削除）後はダウンロード不可。→ [解約FAQ](https://biz.moneyforward.com/support/plan/faq/te07.html)。**保存義務は自事務所に残る**ため、解約・移行時の一括エクスポート手段を事前に確認し、運用手順に落とす |
| 2 | **紙面フォールバック分** | 相手方が電子署名を拒否した場合の紙契約は、MFクラウド契約では**スキャナ保存に対応していない**ため保存要件を満たせない。紙は紙で保存するか、別サービス（クラウドBox等）を検討する |
| 3 | **重要事項説明書の保管** | 重説書面は契約書とは別の書面で、締結前に交付する（署名対象ではない）。MFに登録するのか、交付した事実の記録だけを自システムに残すのかを決める。**交付義務の履行を示す証跡は必要**（法24条の7・監督処分の対象） |
| 4 | **失注案件の受領図面** | 契約に至らなければMFへ送られないため、一時領域のTTLで消える。見積のやり直しや紛争時に参照したいケースがないかを運用で確認する |

