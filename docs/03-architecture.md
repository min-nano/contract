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
│ (PostgreSQL)   │  │ (図面/契約PDF)  │  │ Manager      │  │ MF / Stripe  │
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
Client（発注者）
 ├─ id, kind(individual|corporation), name, kana, address, email, tel
 └─ is_consumer（消費者契約法・特商法の適用判定に使う）

Project（案件）
 ├─ id, client_id, name
 ├─ building（建物の概要：用途、構造、規模、階数、延べ面積、所在地、工事種別）
 └─ documents[]（受領資料）

Document（資料・成果物）
 ├─ id, project_id, kind(received_drawing|quote|important_matters|contract|executed_contract|certificate)
 ├─ storage_uri, filename, content_type, byte_size, page_count
 ├─ sha256（受領時ハッシュ。以後不変）
 └─ received_at, uploaded_by

ContractType（契約種別マスタ）
 ├─ code: design | construction_supervision | seismic_diagnosis
 │        | seismic_retrofit_design | inspection | procedure_agency | other
 ├─ requires_mutual_delivery（法22条の3の3の適用有無。義務 → 締結をブロックする）
 ├─ requires_important_matters（法24条の7の適用有無。義務 → 締結をブロックする）
 ├─ recommends_quote（法24条の10の対象か。努力義務 → 警告のみ。ブロックしない）
 ├─ fee_standard: kokuji8 | kokuji670 | none（適用する業務報酬基準）
 ├─ template_id
 └─ input_schema（Zodスキーマ識別子）

Contract（契約）
 ├─ id, project_id, contract_type_code, status, version, parent_contract_id
 ├─ terms: jsonb（法定記載事項を含む契約条件のスナップショット）
 ├─ fee_amount, fee_tax, payment_schedule[]
 ├─ attached_document_ids[]（別紙目録）
 ├─ esign_provider, esign_document_id, executed_at, executed_pdf_document_id
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

**3つのフラグは強制の強さが違う。混ぜないこと**（→ [01-legal-requirements.md §5](01-legal-requirements.md)）。

| フラグ | 根拠 | 違反時の扱い |
| --- | --- | --- |
| `requires_mutual_delivery` | 法22条の3の3（義務・監督処分あり） | **409 を返して締結を止める** |
| `requires_important_matters` | 法24条の7（義務・監督処分あり） | **409 を返して締結を止める** |
| `recommends_quote` | 法24条の10（努力義務・監督処分なし） | **止めない。** 警告を返して続行できる |

3つのフラグの対象範囲は一致しない。とくに `recommends_quote` は
書面契約義務が及ばない調査・耐震診断にも立つ。「だいたい同じだから」で1つに畳まない。

### 報酬算定は業務報酬基準の告示ごとに分ける

`fee_standard` で選択する。告示第8号と告示第670号は**費目の構成が違う**
（第670号には検査費がある）ため、1つの計算器に押し込めない。
→ [01-legal-requirements.md §1.7](01-legal-requirements.md)

### 適用法令バージョン

改正法附則第4条により、法22条の3の3は**施行日以後に締結される契約**に適用される。
`Contract` は締結日を保持し、そこから適用法令バージョン（改正前／改正後）を判定する。
施行日をまたぐ案件があるため、この判定をハードコードした定数で分岐させない。

## 4. ストレージ方針 — Google Drive を使うべきか

**結論: アプリケーションのデータストアとしては Google Drive を使わない。Cloud Storage を使う。**
Drive は「人が図面をやりとりする作業場所」としてのみ残し、そこから GCS へ一方向で取り込む。

→ 決定の詳細は [adr/0003-object-storage-gcs-over-google-drive.md](adr/0003-object-storage-gcs-over-google-drive.md)

要点:

| 観点 | Google Drive | Cloud Storage |
| --- | --- | --- |
| 共有リンクの誤設定リスク | 「リンクを知っている全員」に**UIから1クリックで**変更できてしまう。契約書・図面には致命的 | 均一なバケットレベルアクセス（uniform bucket-level access）＋公開アクセス防止を組織ポリシーで強制でき、意図しない公開が構造的に起きない |
| 一時アクセスの発行 | 権限付与はアカウント単位。アカウントを持たない発注者への安全な一時公開が苦手 | **署名付きURL**（有効期限つき）で発注者に直接渡せる。今回の要件と合致 |
| 監査ログ | Workspace 監査ログ。アプリ操作との突合が弱い | Cloud Audit Logs でアプリ・IAMと同一基盤で追跡できる |
| 暗号鍵 | 顧客管理鍵の適用範囲が限定的 | **CMEK**（Cloud KMS）でバケット単位に顧客管理鍵を適用できる |
| 保持・改ざん防止 | ゴミ箱からの復元・オーナー変更などのゆらぎ | **バケットロック（保持ポリシー）＋オブジェクトバージョニング**で、電子帳簿保存法の保存要件に対応しやすい |
| サービスアカウント運用 | 共有ドライブでないとサービスアカウント所有ファイルの扱いが煩雑。個人アカウント退職時の権限喪失リスク | IAMで完結。人の異動と無関係 |
| APIの性格 | ファイル同期・コラボレーション向け。レート制限とクォータがアプリ用途に不向き | オブジェクトストレージとして設計されている |

**セキュリティ上「問題ないか」への回答**: Workspace 自体のセキュリティが不足しているわけではない。
問題は**運用上の事故確率**である。契約書・図面・建築主の個人情報を扱う以上、
「共有設定を1クリック間違えると外部公開される」経路をアーキテクチャから排除する価値が高い。

**必要性への回答**: 現時点で Drive でなければ実現できない要件はない。
Workspace を契約済みであることは、Drive をアプリのストレージにする理由にはならない。
ただし次の用途では Drive を使う:

- 発注者から**メール添付で送られてきた図面を、人が受け取って整理する場所**（既存業務の継続）
- 事務所内での作図中ファイルの共同編集

この Drive 上の「受領フォルダ」から本システムへ取り込む導線（Drive Picker またはアップロード）を用意し、
**取り込んだ時点で GCS にコピーしてハッシュを固定**する。以後、契約に添付されるのは GCS 上の不変オブジェクトのみ。

### GCS のバケット設計

| バケット | 用途 | 設定 |
| --- | --- | --- |
| `-received` | 受領資料 | バージョニング有効、CMEK、保持ポリシー10年 |
| `-generated` | 生成した見積書・重説書面・契約書ドラフト | バージョニング有効、CMEK |
| `-executed` | 締結済PDF・合意締結証明書 | **バケットロックによる保持ポリシー10年**、CMEK、削除禁止 |

すべて uniform bucket-level access、公開アクセス防止（組織ポリシー `storage.publicAccessPrevention`）を強制。
発注者への配布は署名付きURL（有効期限15分）で行う。

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

電子取引データ（見積書・契約書・請求書）は次を満たして保存する。

- **真実性**: 締結済PDFはMFの長期署名（タイムスタンプ付き）で担保。
  自システム保管分は `-executed` バケットの保持ポリシーで訂正・削除を防止する。
  訂正が必要な場合は変更契約として新バージョンを作成し、元データは残す。
- **可視性**: 「取引年月日」「取引金額」「取引先」で検索できること。
  `contracts` テーブルに `executed_at` / `fee_amount` / `client_id` のインデックスを張り、
  管理画面に3項目の複合検索と範囲検索を実装する。ダウンロード可能な形式で提示すること。
