# 戦術的設計のベストプラクティス

戦術的設計を実践する際の推奨パターンとアンチパターンをまとめます。

## 目次
1. [ユビキタス言語をコードに反映する](#ユビキタス言語をコードに反映する)
2. [ビジネスルールをコードで表現する](#ビジネスルールをコードで表現する)
3. [値オブジェクトのベストプラクティス](#値オブジェクトのベストプラクティス)
4. [エンティティのベストプラクティス](#エンティティのベストプラクティス)
5. [集約のベストプラクティス](#集約のベストプラクティス)
6. [リポジトリのベストプラクティス](#リポジトリのベストプラクティス)
7. [アプリケーションサービスのベストプラクティス](#アプリケーションサービスのベストプラクティス)
8. [パッケージ構成と依存関係制御](#パッケージ構成と依存関係制御)
9. [テスト容易性と変更影響範囲の制限](#テスト容易性と変更影響範囲の制限)
10. [一般的なアンチパターン](#一般的なアンチパターン)
11. [まとめ](#まとめ)

---

## ユビキタス言語をコードに反映する

**ドメインエキスパートと開発者が同じ言葉を使い、その言葉をそのままコードに反映する**。

### なぜ重要なのか？

ユビキタス言語をコードに反映することで、以下のメリットが得られます：

| メリット | 説明 |
|---------|------|
| **コミュニケーションコスト削減** | ドメインエキスパートと開発者が「翻訳」なしで会話できる |
| **誤解の防止** | 「ユーザー」「カスタマー」「会員」の混用を避ける |
| **コードの可読性向上** | 新しいメンバーが理解しやすい |
| **要件変更への追従性** | 業務側の変更がコードに直結する |

> 💡 **ポイント**
> ドメインエキスパートが「注文を確定する」と言ったら、コードでも `order.confirm()` と書く。
> `order.setStatus(1)` や `order.process()` のような技術的な表現は避ける。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **ドメイン用語をそのまま使う** | `order.confirm()` ← 「注文を確定する」 |
| **コード自体が説明する** | コメントなしで意図が伝わる |
| **略語を避ける** | `cust` → `customer`、`qty` → `quantity` |
| **ビジネス用語と一致** | 業務で使う言葉 = コードの名前 |

### コード例

```typescript
// ✅ 良い例：ビジネス用語をそのまま使用
class Order {
  confirm(): void { /* 注文を確定する */ }
  cancel(): void { /* 注文をキャンセルする */ }
  ship(): void { /* 注文を発送する */ }
}

class Customer {
  upgradeToGoldMember(): void { /* ゴールド会員に昇格 */ }
  isEligibleForDiscount(): boolean { /* 割引対象か判定 */ }
}

// コードを読むだけでビジネスの流れがわかる
const order = Order.create(customer.id, items);
if (customer.isEligibleForDiscount()) {
  order.applyDiscount(discountRate);
}
order.confirm();

// ❌ 悪い例：技術用語や略語
class Ord {
  proc(): void { /* 何を処理？ */ }
  updSts(s: number): void { /* 何のステータス？ */ }
}

class Cust {
  chkFlg(): boolean { /* 何のフラグ？ */ }
}
```

### ユビキタス言語チェックリスト

| チェック項目 | 説明 |
|-------------|------|
| クラス名は業務用語か？ | `Order`, `Customer`, `Product` |
| メソッド名はアクションを表すか？ | `confirm()`, `cancel()`, `ship()` |
| コメントなしで理解できるか？ | コード自体がドキュメント |
| ドメインエキスパートに見せて通じるか？ | 非エンジニアでも読める |

### よくある間違いと対処法

| 間違い | 問題点 | 対処法 |
|---------|---------|--------|
| 技術用語を使う | `DataProcessor`, `Handler` | 業務で使う言葉に置き換える |
| CRUD名を使う | `updateOrder()`, `deleteUser()` | `confirm()`, `deactivate()` に |
| 略語を多用 | `calcAmt()`, `procReq()` | フルスペルで書く |
| マジックナンバー | `status === 1` | `OrderStatus.PENDING` を使う |

---

## ビジネスルールをコードで表現する

**ビジネスルールは値オブジェクトや集約メソッドに閉じ込め、コードで明確に表現する**。

### なぜコードで表現するのか？

ビジネスルールをコードに埋め込むことで：

| メリット | 説明 |
|---------|------|
| **ルール違反をコンパイル時に検出** | 型システムがガード |
| **ルールの分散を防止** | 1箇所に集約で変更容易 |
| **テスト可能** | ルールをユニットテストで検証 |
| **ドキュメント化** | コードが仕様書になる |

> ⚠️ **注意**
> ビジネスルールをコントローラーやサービス層に書くと、分散して管理が困難になる。
> 必ずドメイン層（値オブジェクト、集約、ドメインサービス）に配置する。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **値オブジェクトにバリデーション** | `Email`、`Money` でドメインルールを強制 |
| **集約メソッドでルール表現** | `order.addItem()` で品目数上限をチェック |
| **仕様パターンで複雑なルール** | `PremiumCustomerSpec` で判定ロジックを分離 |
| **ドメインイベントで副作用** | `OrderConfirmed` で在庫引当をトリガー |

### コード例

```typescript
// ✅ 値オブジェクトでドメインルールを強制
class Quantity {
  constructor(private readonly value: number) {
    // ビジネスルール：数量は1以上100以下
    if (value < 1 || value > 100) {
      throw new Error('数量は1〜100の範囲で指定してください');
    }
  }
}

class Email {
  constructor(private readonly value: string) {
    // ビジネスルール：メールアドレスの形式
    if (!this.isValidFormat(value)) {
      throw new Error('不正なメールアドレス形式です');
    }
  }
}

// ✅ 集約メソッドでビジネスルールを表現
class Order {
  // ビジネスルール：1注文は10品目まで
  addItem(productId: ProductId, quantity: Quantity): void {
    if (this.items.length >= 10) {
      throw new Error('1注文につき10品目までです');
    }
    this.items.push(new OrderItem(productId, quantity));
  }

  // ビジネスルール：発送済み注文はキャンセル不可
  cancel(): void {
    if (this.status === OrderStatus.SHIPPED) {
      throw new Error('発送済みの注文はキャンセルできません');
    }
    this.status = OrderStatus.CANCELLED;
  }
}
```

### ルールの配置場所

| ルールの種類 | 配置場所 | 例 |
|-------------|---------|----|
| 単一の値の制約 | 値オブジェクト | 金額は0以上、メール形式 |
| 集約内の整合性 | 集約メソッド | 注文は10品目まで |
| 複数集約にまたがる | ドメインサービス | 在庫確認 + 注文作成 |
| 複雑な判定ロジック | 仕様パターン | プレミアム会員の条件 |

---

## 値オブジェクトのベストプラクティス

値オブジェクトは**属性の値そのものに意味があるオブジェクト**です。IDではなく、値で等価性を判断します。

### なぜ値オブジェクトを使うのか？

| 問題 | プリミティブ型 | 値オブジェクト |
|------|-------------|---------------|
| バリデーション | 呼び出し側でチェック | コンストラクタで強制 |
| 読みやすさ | `price: number` は意味不明 | `price: Money` は明確 |
| ロジックの分散 | 計算ロジックが分散 | クラス内に集約 |
| 型安全性 | `userId` と `orderId` が混在 | 型で区別可能 |

> 💡 **実践のコツ**
> 「この値に制約はあるか？」「この値に対する操作はあるか？」と問いかけてみてください。
> 答えが Yes なら、値オブジェクトにする候補です。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **プリミティブ型を避ける** | `string` ではなく `Email` クラスを使う |
| **不変にする** | `readonly` で変更を防ぐ |
| **バリデーションはコンストラクタ** | 生成時に必ず検証 |
| **操作は新しいインスタンスを返す** | `add()` は新しい `Money` を返す |

### コード例

```typescript
// ✅ 良い例：値オブジェクトを使う
class Money {
  constructor(private readonly amount: number) {
    if (amount < 0) {
      throw new Error('金額は0以上');
    }
  }

  add(other: Money): Money {
    return new Money(this.amount + other.amount); // 新しいインスタンス
  }

  equals(other: Money): boolean {
    return this.amount === other.amount;
  }
}

// ❌ 悪い例：プリミティブ型をそのまま使う
function createOrder(price: number) {  // 負の値が通ってしまう
  // ...
}
```

### 値オブジェクトにすべき候補

| 候補 | 理由 |
|-----|------|
| メールアドレス | フォーマット検証が必要 |
| 電話番号 | 形式チェックが必要 |
| 金額 | 計算ロジック + 負の値チェック |
| 住所 | 複数の値をセットで扱う |
| 期間 | 開始日 <= 終了日の検証 |

---

## エンティティのベストプラクティス

エンティティは**一意の識別子（ID）を持ち、ライフサイクルを通じて属性が変化しうるオブジェクト**です。

### 値オブジェクトとの違い

| 特徴 | 値オブジェクト | エンティティ |
|------|------------------|---------------|
| 同一性 | 値で判断 | IDで判断 |
| 可変性 | 不変 | 可変（属性が変化） |
| ライフサイクル | なし（使い捨て） | あり（追跡可能） |
| 例 | Email, Money, Address | User, Order, Product |

> 💡 **判断のコツ**
> 「同じ名前、同じ年齢の人は同一人物か？」→ No（IDが必要）→ エンティティ
> 「100円は別の100円と区別する必要があるか？」→ No → 値オブジェクト

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **IDで同一性を判断** | 属性が変わってもIDが同じなら同一 |
| **状態変更はメソッド経由** | setterを公開しない |
| **不変条件を守る** | 常に有効な状態を保つ |
| **ファクトリメソッドで生成** | `Order.create()` でバリデーション |

### なぜsetterを公開しないのか？

```typescript
// ✗ setterを公開すると...
order.status = 'CONFIRMED';  // 任意の値が設定可能
order.status = 'INVALID';    // 不正な状態遺移も可能

// ✓ メソッド経由なら...
order.confirm();  // ビジネスルールを強制できる
// 内部で PENDING → CONFIRMED の遺移のみ許可
```

### コード例

```typescript
// ✅ 良い例
class Order {
  private constructor(
    private readonly id: OrderId,
    private status: OrderStatus,
    private items: OrderItem[]
  ) {}

  // ファクトリメソッド
  static create(customerId: CustomerId, items: OrderItemInput[]): Order {
    if (items.length === 0) {
      throw new Error('注文には1つ以上の商品が必要');
    }
    return new Order(OrderId.generate(), OrderStatus.PENDING, []);
  }

  // 状態変更はメソッド経由
  confirm(): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new Error('確定できない状態');
    }
    this.status = OrderStatus.CONFIRMED;
  }

  // IDで比較
  equals(other: Order): boolean {
    return this.id.equals(other.id);
  }
}

// ❌ 悪い例：setterを公開
class BadOrder {
  public status: string;  // 誰でも変更可能
  public items: any[];    // 不整合な状態になりうる
}
```

---

## 集約のベストプラクティス

集約は**データの整合性を保証する境界**です。トランザクションの単位として設計します。

### 集約の役割

```
┌────────────────────────────────────────────────┐
│  集約の境界内で保証されること                      │
│                                                │
│  1. トランザクションの原子性（全か無か）              │
│  2. ビジネスルールの強制（不変条件）                │
│  3. 内部エンティティ間の整合性                    │
└────────────────────────────────────────────────┘
```

### なぜ集約を小さく保つのか？

| 大きな集約の問題 | 小さな集約のメリット |
|----------------|--------------------|
| ロック競合が頻発 | 並行性が向上 |
| メモリ消費大 | 軽量で高速 |
| テストが困難 | テストが容易 |
| 変更影響大 | 変更影響が限定的 |

> ⚠️ **警告**
> 「Order が Customer を持ち、Customer が Address を持つ」のような巨大集約は避ける。
> Order は CustomerId だけを持ち、必要ならリポジトリ経由で取得する。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **小さく保つ** | 1集約 = 1トランザクション |
| **集約ルート経由でアクセス** | 内部エンティティを直接操作しない |
| **他の集約はIDで参照** | オブジェクト参照ではなくIDのみ |
| **結果整合性を活用** | 集約間は非同期で整合性をとる |

### コード例

```typescript
// ✅ 良い例：他の集約はIDで参照
class Order {
  constructor(
    private readonly id: OrderId,
    private readonly customerId: CustomerId,  // ← IDのみ参照
    private items: OrderItem[]
  ) {}

  // 集約ルート経由でのみ操作
  addItem(productId: ProductId, quantity: Quantity): void {
    if (this.items.length >= 10) {
      throw new Error('10品目まで');
    }
    this.items.push(new OrderItem(productId, quantity));
  }
}

// ❌ 悪い例：オブジェクト参照を持つ
class BadOrder {
  constructor(
    private customer: Customer,  // ← オブジェクト全体を参照
    private items: OrderItem[]
  ) {}
}
```

### 集約設計のガイドライン

```
┌────────────────────────────────────────────────────┐
│  1つの集約 = 1つのトランザクション                  │
│                                                     │
│  ✅ 小さい集約:                                     │
│     Order → OrderItem（直接所有）                  │
│                                                     │
│  ❌ 大きすぎる集約:                                 │
│     Order → Customer → Address → ...              │
│     （変更が伝播しすぎる）                          │
└────────────────────────────────────────────────────┘
```

---

## リポジトリのベストプラクティス

リポジトリは**集約の永続化を抽象化するパターン**です。ドメイン層から見ると「コレクション」のように見えます。

### リポジトリの役割

| 役割 | 説明 |
|------|------|
| **永続化の隠蔽** | DBの詳細をドメイン層から隠す |
| **集約単位の操作** | バラバラに保存しない |
| **テスト容易性** | モック実装に差し替え可能 |
| **インフラ切り替え** | DB変更時もドメイン層は不変 |

> 💡 **ポイント**
> リポジトリのインターフェースはドメイン層に定義し、実装はインフラ層に置く。
> これが「依存関係逆転の原則（DIP）」の実践。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **集約ルートごとに1つ** | `OrderRepository`, `CustomerRepository` |
| **コレクション風API** | `find`, `save`, `remove` |
| **ドメイン層にインターフェース** | 実装はインフラ層 |
| **集約全体を取得/保存** | 部分的な操作は避ける |

### コード例

```typescript
// 📦 domain/repositories.ts（インターフェース）
interface OrderRepository {
  findById(id: OrderId): Order | null;
  findByCustomerId(customerId: CustomerId): Order[];
  save(order: Order): void;
  remove(order: Order): void;
}

// 🗄️ infrastructure/postgres-order-repository.ts（実装）
class PostgresOrderRepository implements OrderRepository {
  save(order: Order): void {
    // ✅ 集約全体を保存
    const row = order.toPersistence();
    db.query(`INSERT INTO orders ...`, row);
    
    for (const item of order.items) {
      db.query(`INSERT INTO order_items ...`, item.toPersistence());
    }
  }
}
```

### ❌ アンチパターン

```typescript
// ❌ 悪い例：内部エンティティ単独のリポジトリ
interface OrderItemRepository {  // これは作らない
  findById(id: OrderItemId): OrderItem;
  save(item: OrderItem): void;
}
```

---

## アプリケーションサービスのベストプラクティス

アプリケーションサービス（UseCase）は**ユースケースの実行を調整する司令塔**です。

### アプリケーション層の責務

| すべきこと | すべきでないこと |
|-------------|----------------|
| リポジトリの呼び出し | ビジネスルールの実装 |
| トランザクション境界の管理 | 値の計算ロジック |
| ドメインオブジェクトの操作呼び出し | 条件分岐の複雑なロジック |
| DTOへの変換 | ドメインイベントの発行処理 |

> ⚠️ **警告**
> アプリケーションサービスが肥大化したら、ドメイン層へのロジック移動を検討。
> if文やループが増えたら危険信号。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **薄く保つ** | ビジネスロジックはドメイン層へ |
| **調整役に徹する** | リポジトリ呼び出し + ドメイン操作 |
| **トランザクション境界** | ここで管理 |
| **DTOへの変換** | プレゼンテーション層向けに変換 |

### コード例

```typescript
// ✅ 良い例：薄いアプリケーションサービス
class CreateOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private customerRepo: CustomerRepository
  ) {}

  execute(input: CreateOrderInput): OrderId {
    // 1. 取得
    const customer = this.customerRepo.findById(input.customerId);
    
    // 2. ドメイン操作（ロジックはOrder内）
    const order = Order.create(customer.id, input.items);
    
    // 3. 保存
    this.orderRepo.save(order);
    
    return order.id;
  }
}

// ❌ 悪い例：ロジックがアプリケーション層に
class BadCreateOrderUseCase {
  execute(input: CreateOrderInput): OrderId {
    // ❌ ビジネスロジックがここに
    if (input.items.length > 10) {
      throw new Error('10品目まで');
    }

    let total = 0;
    for (const item of input.items) {
      total += item.price * item.quantity;  // ❌ 計算ロジックがここに
    }

    if (total > 10000) {
      total *= 0.9;  // ❌ 割引ルールがここに
    }

    // ...
  }
}
```

---

## パッケージ構成と依存関係制御

**パッケージ構成でコンテキストを分断し、インターフェースで依存関係を制御する**。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **コンテキストごとにパッケージ分割** | `order/`, `customer/`, `inventory/` |
| **依存関係逆転の原則（DIP）** | ドメイン層がインフラ層に依存しない |
| **インターフェースで詳細を隠蔽** | `OrderRepository` interface |
| **集約境界で変更影響範囲を制限** | 集約内で変更を閉じ込める |

### ディレクトリ構成例

```
src/
├── order/                    # 注文コンテキスト
│   ├── domain/
│   │   ├── Order.ts          # 集約ルート
│   │   ├── OrderItem.ts      # 集約内エンティティ
│   │   ├── Money.ts          # 値オブジェクト
│   │   └── OrderRepository.ts # インターフェース
│   ├── application/
│   │   └── CreateOrderUseCase.ts
│   └── infrastructure/
│       └── PostgresOrderRepository.ts
│
├── customer/                 # 顧客コンテキスト
│   ├── domain/
│   ├── application/
│   └── infrastructure/
│
└── inventory/                # 在庫コンテキスト
    ├── domain/
    ├── application/
    └── infrastructure/
```

### 依存関係逆転（DIP）

```typescript
// 📦 domain/OrderRepository.ts（インターフェース）
// ドメイン層で定義 → 依存関係は抽象に向かう
export interface OrderRepository {
  findById(id: OrderId): Order | null;
  save(order: Order): void;
}

// 🗄️ infrastructure/PostgresOrderRepository.ts（実装）
// インフラ層がドメイン層のインターフェースに依存
import { OrderRepository } from '../domain/OrderRepository';

export class PostgresOrderRepository implements OrderRepository {
  findById(id: OrderId): Order | null {
    // SQL実装
  }
}

// 🎯 application/CreateOrderUseCase.ts
// アプリケーション層はインターフェースに依存（実装を知らない）
export class CreateOrderUseCase {
  constructor(private orderRepo: OrderRepository) {} // ← interface
}
```

### 依存関係の方向

```
┌─────────────────────────────────────────────────────────┐
│                                                          │
│  インフラ層 ──実装──→ ドメイン層（interface）            │
│       ↑                        ↑                        │
│       │                        │                        │
│  実装の詳細を隠蔽         ビジネスロジックに集中          │
│                                                          │
│  ──────────────────────────────────────────────────────  │
│                                                          │
│  ❌ 悪い例：ドメイン層 → インフラ層                      │
│     Order が PostgresOrderRepository に依存              │
│     → DBを変えると Order も変更が必要                    │
│                                                          │
│  ✅ 良い例：インフラ層 → ドメイン層                      │
│     PostgresOrderRepository が Order に依存              │
│     → DBを変えても Order は変更不要                      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## テスト容易性と変更影響範囲の制限

**ドメインルールをテストし、集約設計で変更の影響範囲を制限する**。

### なぜテスト容易性が重要なのか？

| 観点 | 説明 |
|------|------|
| **リグレッション防止** | 既存ルールが壊れていないことを確認 |
| **仕様のドキュメント化** | テストコードが仕様書の役割 |
| **リファクタリング支援** | テストがあれば安心して変更可能 |
| **設計の品質向上** | テストしやすいコード = 良い設計 |

> 💡 **ポイント**
> ドメイン層は外部依存が少ないため、ユニットテストが書きやすい。
> これはDDDの大きなメリットの一つ。

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **ドメインルールをユニットテスト** | 集約・値オブジェクトの振る舞いをテスト |
| **インターフェースでモック化** | リポジトリをモックしてテスト容易に |
| **集約境界で影響範囲制限** | 変更は集約内に閉じ込める |
| **依存関係逆転で結合度低下** | 実装を差し替え可能に |

### ドメインルールのテスト

```typescript
// ✅ ビジネスルールをテスト
describe('Order', () => {
  describe('addItem', () => {
    it('10品目まで追加できる', () => {
      const order = Order.create(customerId, []);
      for (let i = 0; i < 10; i++) {
        order.addItem(productId, new Quantity(1));
      }
      expect(order.itemCount()).toBe(10);
    });

    it('11品目目はエラー', () => {
      const order = Order.create(customerId, []);
      for (let i = 0; i < 10; i++) {
        order.addItem(productId, new Quantity(1));
      }
      expect(() => order.addItem(productId, new Quantity(1)))
        .toThrow('1注文につき10品目までです');
    });
  });

  describe('cancel', () => {
    it('発送済みの注文はキャンセルできない', () => {
      const order = Order.createShipped(customerId, items);
      expect(() => order.cancel())
        .toThrow('発送済みの注文はキャンセルできません');
    });
  });
});

// ✅ 値オブジェクトのテスト
describe('Quantity', () => {
  it('1未満はエラー', () => {
    expect(() => new Quantity(0)).toThrow();
  });

  it('100超はエラー', () => {
    expect(() => new Quantity(101)).toThrow();
  });
});
```

### モックを使ったユースケーステスト

```typescript
// ✅ リポジトリをモックしてユースケースをテスト
describe('CreateOrderUseCase', () => {
  it('注文を作成して保存する', () => {
    // モックリポジトリ
    const mockOrderRepo: OrderRepository = {
      findById: jest.fn(),
      save: jest.fn(),
    };
    const mockCustomerRepo: CustomerRepository = {
      findById: jest.fn().mockReturnValue(customer),
    };

    const useCase = new CreateOrderUseCase(
      mockOrderRepo,
      mockCustomerRepo
    );

    useCase.execute({ customerId: '123', items: [...] });

    // 保存が呼ばれたことを検証
    expect(mockOrderRepo.save).toHaveBeenCalled();
  });
});
```

### テスト戦略

| テスト種類 | 対象 | 目的 |
|-----------|------|------|
| **ユニットテスト** | 値オブジェクト、エンティティ | ビジネスルールの検証 |
| **ユニットテスト** | 集約 | 整合性ルールの検証 |
| **ユニットテスト** | ドメインサービス | 複数集約の協調動作 |
| **結合テスト** | ユースケース + モックリポジトリ | ユースケースフローの検証 |
| **E2Eテスト** | API全体 | システム全体の動作確認 |

---

## 一般的なアンチパターン

### 🚫 避けるべきパターン

| アンチパターン | 問題点 | 解決策 |
|--------------|--------|--------|
| **貧血ドメインモデル** | エンティティがデータのみ持ち、ロジックがサービスに | ロジックをエンティティに移動 |
| **神オブジェクト** | 1つのクラスが全てを担う | 責務ごとにクラス分割 |
| **プリミティブ執着** | `string`, `number` を多用 | 値オブジェクトに置き換え |
| **巨大な集約** | 集約が大きすぎる | 小さな集約 + ID参照に分割 |
| **トランザクションスクリプト** | 手続き型のサービス | ドメインモデルパターンに移行 |

### 貧血ドメインモデル（Anemic Domain Model）

```typescript
// ❌ 貧血ドメインモデル
class Order {
  public id: string;
  public status: string;
  public items: OrderItem[];
  // ロジックなし、データだけ
}

class OrderService {
  confirm(order: Order): void {
    if (order.status !== 'PENDING') {  // ロジックがサービスに
      throw new Error('確定できない');
    }
    order.status = 'CONFIRMED';
  }
}

// ✅ リッチドメインモデル
class Order {
  private status: OrderStatus;

  confirm(): void {  // ロジックがエンティティ内に
    if (this.status !== OrderStatus.PENDING) {
      throw new Error('確定できない');
    }
    this.status = OrderStatus.CONFIRMED;
  }
}
```

---

## まとめ

> 📎 **図解**: [Excalidrawで見る](../other/12_tactical_design_best_practices/best_practices.excalidraw)

### チェックリスト

| カテゴリ | チェック項目 |
|---------|-------------|
| **値オブジェクト** | プリミティブ型の代わりに使っているか？ |
| **エンティティ** | setter を公開せず、メソッド経由で状態変更しているか？ |
| **集約** | 小さく保ち、他の集約はIDで参照しているか？ |
| **リポジトリ** | 集約ルートごとに1つか？インターフェースはドメイン層か？ |
| **アプリケーションサービス** | 薄く保ち、ロジックはドメイン層に委譲しているか？ |

### 設計原則

```
┌─────────────────────────────────────────────────┐
│  1. ドメインロジックはドメイン層に              │
│  2. アプリケーション層は調整役に徹する          │
│  3. 集約は小さく、IDで参照                      │
│  4. 値オブジェクトを積極的に使う                │
│  5. リポジトリは集約単位                        │
└─────────────────────────────────────────────────┘
```

---

## 参考リンク

- [10_tactical_design.md](./10_tactical_design.md) - 戦術的設計の概要
- [11_tactical_components.md](./11_tactical_components.md) - 戦術的設計の構成要素
- [01_ddd_introduction.md](./01_ddd_introduction.md) - DDD入門
