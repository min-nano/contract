# システム構成

**本システムは、社内ポータル（[portal](https://github.com/min-nano/portal)）のツールとして実装する**
（→ [adr/0008](adr/0008-implement-as-portal-tool.md)）。
独自のインフラは持たない。

---

## 1. 構成

```
ブラウザ（Firebase Hosting）
  │   Authorization: Bearer <Clerk セッション JWT>
  ▼
Firebase Hosting ── /api/** リライト ──▶ Cloud Run (portal-api / FastAPI)
                                          │ 1. Clerk JWKS で JWT 検証 → email 確定
                                          │ 2. 共有設定（雛形の場所・報酬の単価）を Firestore から取得
                                          │ 3. email の代理トークン（domain-wide delegation）で
                                          │    Drive / Docs を操作
                                          │ 4. 報酬算定は core/ の .wasm（画面と同じ実装）
                                          ▼
                              Google Workspace の Drive / Docs、Firestore
```

**portal に既にあるものだけで足りる。**
新しく足すのは、契約書ツールの画面・API・雛形マッピング・報酬算定である。

| 追加するもの | 置き場所 |
| --- | --- |
| 画面 | `frontend/tools/contract-formatter/`、`frontend/src/contract-formatter/` |
| API | `/api/tools/contract-formatter/**` |
| 雛形マッピング（単一の情報源） | `backend/app/contract_mapping.json` |
| 生成・解析 | `backend/app/contract.py` |
| 報酬算定 | `core/src/fee.rs`、`core/src/fee_kokuji8.rs`、`core/src/fee_kokuji670.rs` |

### 持たないもの

**[adr/0007](adr/0007-document-preparation-only.md) により、次はすべて存在しない。**

| 持たないもの | 代わりに |
| --- | --- |
| データベース（Cloud SQL） | **PDFの文書情報に入力を埋め込む**（→ §3） |
| オブジェクトストレージ（Cloud Storage） | 生成したPDFは Drive へ保存するか、ブラウザへ返す |
| 発注者向け画面・署名付きトークン・ワンタイムコード | 交付はメール（→ [07-operations.md](07-operations.md)） |
| 決済（Stripe）・Webhook | 請求はマネーフォワード クラウド請求書 |
| Cloud Tasks / Cloud Scheduler（期限アラート） | 手順はチェックリストで守る |
| 電子契約サービスとの連携（`EsignProvider`） | 用いない（→ [adr/0002](adr/0002-esign-provider-abstraction.md) は廃止） |

---

## 2. 認証・認可

**事務所のスタッフだけが使う。** portal の仕組みをそのまま使う。

| 段 | 内容 |
| --- | --- |
| 1 | **Clerk のセッション JWT を検証**（JWKS で署名確認、`exp` / `iss` / `azp`）。トークンからメールアドレスを取り出し、許可ドメイン以外は 403 |
| 2 | **代理アクセストークン**（domain-wide delegation）。確認したメールアドレスのユーザーとして Drive / Docs を呼ぶ |

**雛形の取得も保存も、常に実行ユーザー本人の権限の範囲で行われる。**
本人が読めない雛形・書けないフォルダは扱えない。

> **契約書という書類の重さは、アプリケーションを分けることではなく
> 保存先フォルダの共有設定で担保する**（→ [07-operations.md §4.3](07-operations.md)、
> [adr/0008](adr/0008-implement-as-portal-tool.md)）。

必要な Drive のスコープは portal と同じ。

| スコープ | 使う場面 |
| --- | --- |
| `drive.readonly` | 雛形の取得、編集するPDFの読み込み、Google Picker へ渡すトークン |
| `drive` | PDF の Drive への保存 |
| `documents` | 雛形（Google ドキュメント）のプレースホルダー置換 |

---

## 3. データの持ち方 — **PDFが保存形式である**

**DBを持たない。契約の内容はPDFの文書情報に埋め込む。**

```
フォーム入力 ──→ 雛形の複製に置換 ──→ PDF書き出し ──→ 入力を文書情報へ埋め込む
                                                          │
                                                          ▼
                                                     Drive へ保存
                                                          │
     フォームへ完全復元 ◀── 文書情報を読む ◀── PDFを開く ─┘
```

この設計が満たすもの:

| 要件 | どう満たすか |
| --- | --- |
| **締結時点の契約条件を固定する** | 入力そのものがPDFに入る。**マスタを参照しないので、事務所名や登録番号を更新しても過去の契約書の再現性は壊れない**（現行設計が `Contract.terms` で守ろうとしていたもの） |
| **変更契約を原契約から作る** | 原契約PDFを開けば入力が復元される（→ [05-documents.md §5](05-documents.md)） |
| **二重管理をしない** | 成果物が1つ（PDF）だけになる。DBとファイルの整合性を心配しなくてよい |
| **本ツール以外で作ったPDFも扱える** | 本文のレイアウトから推定して読み込む（portal の証明書ツールと同じ。推定であることを画面に出す） |

**埋め込む内容**

```json
{
  "documentKind": "contract | important_matters | amendment",
  "fields":  { ... 法定記載事項と任意項目 ... },
  "choices": { ... 契約種別・事務所の別・資格区分 ... },
  "fee":     { ... 報酬算定の入力と結果 ... },
  "templateRevision": "2027-01-15T09:12:33Z",
  "generatedAt": "2027-04-15T10:00:00Z"
}
```

- 文書情報は **ASCII に収まるようJSON文字列としてエスケープして持つ**（portal の作法に合わせる）
- **氏名・住所を含む。** 保存先フォルダの共有設定が個人情報の境界になる（→ [07-operations.md §4.3](07-operations.md)）

### 共有設定（Firestore）

portal の `settings_store` を使う。チャンネル（本番／開発／PRプレビュー）で分かれる。

| キー | 内容 |
| --- | --- |
| `template_folder_id` / `template_file_name_contract` | 契約書の雛形 |
| `template_file_name_important_matters` | 重要事項説明書の雛形 |
| `template_file_name_amendment` | 変更契約書の雛形 |
| `template_file_name_fee_breakdown` | 業務報酬内訳書の雛形 |
| `office` | 事務所の情報（名称・所在地・一級/二級/木造の別・開設者名）。**フォームの初期値**であって、印字するのは入力された値 |
| `architects` | 所属建築士（氏名・資格区分・登録番号・管理建築士か）。同上 |
| `fee` | 報酬算定の設定値（→ [06-fee-estimation.md §3](06-fee-estimation.md)） |

> **設定値は「初期値」であって「参照先」ではない。**
> 契約書に印字するのは**フォームに入った値**であり、PDFにもその値が埋まる。
> 設定を後から変えても、過去の契約書は変わらない。

---

## 4. 報酬算定 — 画面とサーバで同じ実装を動かす

portal の必要壁量ツールと同じ作法（→ [06-fee-estimation.md §6](06-fee-estimation.md)）。

- 実装は `core/`（Rust）の1つだけ。**それを .wasm にしたものを画面もサーバも動かす**
- 画面は入力のたびに金額を出す（面積を動かしながら金額を見られる）
- **内訳書を生成するとき、サーバが同じ .wasm で計算し直して突き合わせる。**
  食い違えば警告を出す

---

## 5. セキュリティ

- **PII をログに出さない。** 氏名・住所・メールアドレス・電話番号・建物の所在地。
  portal のロガーの作法に合わせる
- **秘匿情報は環境変数に直書きしない。** portal の既存の扱いに従う
- **アップロードサイズの上限**を設ける（portal のPDFツールと同じ 20MB）
- **雛形フォルダ・契約書類フォルダの共有設定**が実質的な境界である
  （→ [07-operations.md §4.3](07-operations.md)）
- **委託先管理**: Google Workspace / Google Cloud / Clerk を個人情報の委託先として
  一覧化し、安全管理措置の確認記録を残す（個人情報保護法）。
  **Stripe とマネーフォワード クラウド契約は委託先から外れた**（使わないため）

---

## 6. 電子帳簿保存法

**保存は事務所の手元（Drive）で行うため、システムは要件充足の主体にならない。**
真実性は**規則4条1項4号の事務処理規程**、可視性は**小規模事業者の特例＋ファイル名規則**で満たす。
詳細と規程の骨子は [07-operations.md §4.4・§6](07-operations.md)。

**システムができるのは1つだけ** ——
**ファイル名を `YYYYMMDD_取引先名_金額.pdf` の形で既定として組み立てること。**
そのまま保存すれば検索要件の備えになる。

> ⚠ **事務処理規程はリリースのブロッカーである。**
> 規程が未整備のまま運用を始めると、真実性の要件を満たす手段がゼロになる
> （→ [adr/0006](adr/0006-self-hosted-retention.md)）。**規程はコードより先に要る。**
