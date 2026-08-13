# コード規約

このプロジェクトのコードは、**契約という法的効力を持つ文書**を生成し、**金銭を動かす**。
バグは「動かない」ではなく「法定記載事項が欠けた契約書を締結してしまった」「二重に課金した」という形で現れる。
規約はその前提で読むこと。

## 0. 大原則

1. **法令要件はコードで強制する。** 運用でカバーする前提の実装をしない。
   法定必須項目の検証、重説実施のガード、電磁的方法の承諾チェックは、
   UIではなく**サーバ側**で必ず実行する。
2. **監査証跡は消さない。** 契約に関わる状態変化は追記のみ。UPDATE / DELETE で履歴を失わない。
3. **迷ったら安全側に倒す。** 締結を1回止めるコストより、不備のある契約を1件通すコストの方が高い。
4. **既存のコードに合わせる。** ここに書かれていない事柄は、周囲のコードの書き方に従う。

---

## 1. 言語・ツール

| 領域 | 採用 |
| --- | --- |
| 言語 | TypeScript（`strict: true`。フロント・バックとも） |
| ランタイム | Node.js（LTS） |
| APIフレームワーク | Hono |
| バリデーション | Zod |
| ORM | Prisma |
| フロントエンド | React + Vite + TanStack Query |
| テスト | Vitest（＋ Playwright: E2E） |
| Lint / Format | Biome |
| パッケージ管理 | pnpm workspaces |

### モノレポ構成

```
apps/
  api/          Cloud Run にデプロイする API
  web/          Firebase Hosting にデプロイする SPA
packages/
  domain/       ドメインモデル・法令要件のスキーマ（外部依存を持たない）
  contracts/    API の型定義（Zodスキーマ。api と web が共有）
  pdf/          帳票生成
docs/
```

- `packages/domain` は **`node:` 以外の外部ライブラリに依存しない**（Zodのみ許可）。
  法令要件のロジックはここに置き、単体テストで守る。
- `apps/api` から `apps/web` を import しない。逆も同じ。共有は `packages/` を経由する。

---

## 2. TypeScript

- `tsconfig` は `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `noImplicitOverride` を有効にする。
- **`any` 禁止。** 型が不明な値は `unknown` で受け、Zodで絞る。
  やむを得ない場合は `// eslint-disable` ではなく理由をコメントし、レビューで合意する。
- **`as` によるキャスト禁止**（`as const` を除く）。型が合わないのは設計が合っていない兆候。
- 公開関数の引数・戻り値には明示的に型を書く。推論に任せてよいのはローカル変数まで。
- `enum` は使わない。`as const` オブジェクト＋ union 型を使う。

```ts
export const CONTRACT_STATUS = {
  draft: 'DRAFT',
  quoteSent: 'QUOTE_SENT',
  // ...
} as const
export type ContractStatus = (typeof CONTRACT_STATUS)[keyof typeof CONTRACT_STATUS]
```

### 型で不正な状態を作れなくする

契約のステータス遷移や法定必須項目は、実行時チェックだけでなく型でも表現する。

```ts
// 悪い: どの状態でも締結依頼を送れてしまう
function requestSignature(contract: Contract): Promise<void>

// 良い: 締結依頼を送れる状態の契約しか渡せない
function requestSignature(contract: SignatureReadyContract): Promise<void>
```

---

## 3. 命名

- コード上の識別子は**英語**。ドメイン用語は法令上の訳語を統一する（下表）。
- 変数・関数は `camelCase`、型・クラスは `PascalCase`、定数は `SCREAMING_SNAKE_CASE`。
- DBのカラム・テーブルは `snake_case`、テーブル名は複数形。
- 真偽値は `is` / `has` / `can` / `should` で始める。
- **日本語のコメントは可。** むしろ法令の根拠は日本語で書くこと。

### 用語対訳（必ずこれを使う）

| 日本語 | コード上の表記 |
| --- | --- |
| 設計受託契約 | `designContract` / `design` |
| 工事監理受託契約 | `constructionSupervisionContract` / `construction_supervision` |
| 建築主・発注者 | `client` |
| 建築士事務所 | `office` |
| 管理建築士 | `managingArchitect` |
| 建築士 | `architect` |
| 重要事項説明 | `importantMattersExplanation` |
| 電磁的方法による交付の承諾 | `electronicDeliveryConsent` |
| 法定記載事項 | `statutoryTerms` |
| 相互交付 | `mutualDelivery` |
| 締結（済） | `execute` / `executed`（`sign` は個々の署名行為に限る） |
| 手付金 | `deposit` |
| 前払金 | `advancePayment` |
| 業務報酬基準 | `feeStandard` |
| 変更契約 | `amendment` |

`contract` は「契約」全般を指すため、電子契約サービス側の書類を指すときは
`esignDocument` と呼び分ける。

---

## 4. 法令要件の実装ルール

### 4.1 条文の根拠をコードに書く

法令に由来する条件分岐・必須項目には、**必ず条文番号をコメントで残す**。
改正時にどこを直すべきかを追えるようにするため。

```ts
/**
 * 契約書面の法定記載事項（法22条の3の3第1項・規則17条の38）
 * @see docs/01-legal-requirements.md §2
 */
export const statutoryContractTermsSchema = z.object({
  // 法22条の3の3第1項3号: 従事する建築士の氏名・資格区分
  assignedArchitects: z.array(assignedArchitectSchema).min(1),
  // 法22条の3の3第1項4号: 報酬の額及び支払の時期
  feeAmount: z.number().int().positive(),
  paymentSchedule: z.array(paymentTermSchema).min(1),
  // ...
})
```

### 4.2 契約書面と重要事項説明書のスキーマを共通化しない

必須項目セットが異なる（重説の省令事項は規則17条の38の**1号〜6号のみ**）。
共通部分は別スキーマに切り出して合成する。「だいたい同じだから」で1つにまとめない。

```ts
const commonTerms = z.object({ /* 1号〜6号相当 */ })

export const importantMattersSchema = commonTerms          // 法24条の7
export const contractFormSchema = commonTerms.extend({     // 法22条の3の3
  implementationPeriod: periodSchema,   // 規則17条の38第7号
  scopeOfWork: z.string().min(1),       // 規則17条の38第8号
})
```

### 4.3 締結時点の値をスナップショットする

契約書に印字する値は、マスタを参照するのではなく契約レコードにコピーして固定する。
事務所名や建築士の登録番号が後から変わっても、締結済み契約書の再生成結果が変わってはならない。

### 4.4 金額

- **金額は整数（円）で扱う。** `number` の小数、`float` 型のカラムを使わない。
- 消費税の計算は税率ごとに区分して集計する（複数税率・非課税・不課税を扱う）。
- 表示のフォーマットは表示層でのみ行う。ドメイン層は整数のまま扱う。

---

## 5. API 設計

- パスは複数形の名詞。動詞はHTTPメソッドで表す。
  例外として状態遷移は `POST /contracts/{id}/actions/request-signature` のように
  `actions/` 配下に置いてよい。
- リクエスト・レスポンスは `packages/contracts` の Zod スキーマで定義し、
  そこから型を導出する。**手書きの `interface` と実行時検証を二重管理しない。**
- エラーレスポンスは統一形式にする。

```jsonc
{
  "error": {
    "code": "STATUTORY_TERMS_INCOMPLETE",
    "message": "契約書面の法定記載事項が不足しています",
    "details": [{ "field": "assignedArchitects", "reason": "required" }]
  }
}
```

- HTTPステータスは意味に沿って使う。
  状態遷移の前提を満たさない場合は **409 Conflict**（400 で潰さない）。
- **金銭・締結に関わるすべての POST は冪等にする。**
  クライアントが生成した `Idempotency-Key` ヘッダを受け取り、
  同一キーの再送には同じ結果を返す。決済と締結依頼では必須。
- Webhook 受信は署名検証 → イベントID重複排除 → 処理、の順を厳守する。
  処理の途中で失敗したら 5xx を返して再送させる。

---

## 6. データベース

- マイグレーションは Prisma Migrate。**手でDBを触らない。**
- 破壊的変更（カラム削除・リネーム）は2段階でリリースする（追加 → 移行 → 削除）。
- 金額は `Int`（円）。日時は `timestamptz` で保存し、**UTCで保存・JSTで表示**する。
- 契約に関わるテーブルは**論理削除も原則使わない**。
  取り消しは新しいイベント（`ContractEvent`）として追記する。
- `contract_events` はアプリケーション用DBロールに INSERT と SELECT のみ許可し、
  UPDATE / DELETE を権限レベルで禁止する。
- N+1 を作らない。一覧取得は必要な関連を明示的に読み込む。

---

## 7. ログ・監視

- 構造化ログ（JSON）で Cloud Logging に出す。`severity`, `contractId`, `traceId` を必ず含める。
- **個人情報をログに出さない。** 氏名・住所・メールアドレス・電話番号・建物所在地・
  アクセストークン・カード情報は禁止。ID とハッシュで追跡する。
  ロガーは PII フィールド名を自動マスクするラッパーを経由させ、素の `console.log` を禁止する。
- 例外を握り潰さない。`catch` したら必ず「ログに出す」か「上位に再送出する」かのどちらかを行う。
- 次の事象はアラート対象:
  締結依頼の送信失敗、Webhook の連続失敗、決済オーソリの期限切れ間近、
  法定必須項目の検証エラーが本番で発生（＝ドラフト生成側の不具合）。

---

## 8. テスト

- **必ずテストを書く対象**（レビューで通さない基準にする）
  - 法定記載事項のバリデーション（`packages/domain`）
  - 契約のステータス遷移（不正な遷移が弾かれること）
  - 金額・消費税の計算
  - PDF生成の内容（テキスト抽出して法定項目の存在を検証する）
  - Webhook の署名検証と冪等性
- テスト名は日本語で書いてよい。何を保証しているかが伝わることを優先する。

```ts
it('重要事項説明の実施記録がない契約は署名依頼に進めない', async () => { /* ... */ })
```

- 外部サービス（MF・Stripe・Clerk）はアダプタ層で抽象化し、テストではフェイク実装を使う。
  ネットワークを叩くテストは CI から分離する。
- E2E は「見積リンクを開いてから締結まで」のハッピーパスを1本、必ず維持する。

---

## 9. 秘匿情報

- **リポジトリに秘密を置かない。** APIキー、クライアントシークレット、署名鍵、
  Webhook シークレット、本番の顧客データ、実在する図面ファイル。
- 本番の値は Secret Manager。ローカルは `.env.local`（gitignore 済み）。
- `.env.example` にキー名だけを列挙し、値はプレースホルダにする。
- 誤コミットに備えて secret scanning を CI で回す。

---

## 10. Git

- ブランチ: `feat/...`, `fix/...`, `chore/...`, `docs/...`
- コミットメッセージは [Conventional Commits](https://www.conventionalcommits.org/ja/)。
  本文は日本語で構わない。

```
feat(contract): 法定記載事項の検証を追加

法22条の3の3第1項および規則17条の38の各号を必須項目として検証する。
契約種別が設計受託・工事監理受託の場合のみ適用する。
```

- PRは小さく保つ。法令要件に関わる変更は、**根拠条文と docs/ の該当節を PR 本文に書く**。
- `main` への直push禁止。CI（型チェック・Lint・テスト）通過を必須にする。
- 法令要件のロジック（`packages/domain`）と契約書テンプレートの変更は、
  管理建築士のレビューを必須とする（CODEOWNERS で設定する）。

---

## 11. ドキュメント

- 仕様の判断理由は ADR（`docs/adr/`）に残す。「なぜそうしなかったか」を書く。
- 法令の解釈に関わる判断は、必ず `docs/01-legal-requirements.md` に反映してからコードを書く。
- 法改正・省令公布のたびに `docs/01-legal-requirements.md` を再検証し、
  差分を ADR とマイグレーション計画に落とす。
