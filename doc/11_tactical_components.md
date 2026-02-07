# 戦術的設計の構成要素

戦術的設計でドメインモデルを実装するための主要なパターンを解説します。

## 目次
1. [値オブジェクト（Value Object）](#値オブジェクトvalue-object)
2. [エンティティ（Entity）](#エンティティentity)
3. [集約（Aggregate）](#集約aggregate)
4. [リポジトリ（Repository）](#リポジトリrepository)
5. [ドメインサービス（Domain Service）](#ドメインサービスdomain-service)
6. [アプリケーションサービス（Application Service）](#アプリケーションサービスapplication-service)
7. [ドメインイベント（Domain Event）](#ドメインイベントdomain-event)
8. [ファクトリ（Factory）](#ファクトリfactory)
9. [仕様（Specification）](#仕様specification)
10. [まとめ](#まとめ)

---

## 値オブジェクト（Value Object）

**属性の値そのものに意味があるオブジェクト**。同一性ではなく、値で比較される。

### 特徴

| 特徴 | 説明 |
|------|------|
| **不変（Immutable）** | 一度作成したら変更できない |
| **等価性** | 属性の値が同じなら等しい |
| **副作用がない** | 操作しても他に影響しない |
| **交換可能** | 同じ値なら置き換えても問題ない |

### コード例

```typescript
// ❌ プリミティブ型をそのまま使う（悪い例）
function createOrder(price: number, email: string) {
  // price が負の値でも通ってしまう
  // email が不正な形式でも通ってしまう
}

// ✅ 値オブジェクトを使う（良い例）
class Money {
  constructor(private readonly amount: number) {
    if (amount < 0) {
      throw new Error('金額は0以上である必要があります');
    }
  }

  // 不変：新しいインスタンスを返す
  add(other: Money): Money {
    return new Money(this.amount + other.amount);
  }

  // 等価性：値で比較
  equals(other: Money): boolean {
    return this.amount === other.amount;
  }
}

class Email {
  constructor(private readonly value: string) {
    if (!this.isValid(value)) {
      throw new Error('不正なメールアドレスです');
    }
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}
```

### 値オブジェクトの例

| 例 | 説明 |
|----|------|
| `Money` | 金額（amount + currency） |
| `Email` | メールアドレス |
| `Address` | 住所（郵便番号、都道府県、市区町村...） |
| `DateRange` | 期間（開始日〜終了日） |
| `Color` | 色（RGB値） |

---

## エンティティ（Entity）

**一意な識別子（ID）を持ち、ライフサイクルを通じて同一性を保つオブジェクト**。

### 特徴

| 特徴 | 説明 |
|------|------|
| **識別子を持つ** | UUID、連番などで一意に識別 |
| **同一性** | IDが同じなら同一とみなす |
| **可変** | 状態が変化しうる |
| **ライフサイクル** | 作成→更新→削除の流れがある |

### コード例

```typescript
class Order {
  constructor(
    private readonly id: OrderId,      // 識別子
    private customerId: CustomerId,
    private items: OrderItem[],
    private status: OrderStatus        // 状態が変化する
  ) {}

  // IDで同一性を判断
  equals(other: Order): boolean {
    return this.id.equals(other.id);
  }

  // 状態を変更するメソッド
  confirm(): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new Error('確定できない状態です');
    }
    this.status = OrderStatus.CONFIRMED;
  }

  cancel(): void {
    if (this.status === OrderStatus.SHIPPED) {
      throw new Error('発送済みの注文はキャンセルできません');
    }
    this.status = OrderStatus.CANCELLED;
  }
}
```

### 値オブジェクト vs エンティティ

| 観点 | 値オブジェクト | エンティティ |
|:----:|:------------:|:----------:|
| 識別子 | なし | あり |
| 比較方法 | 値で比較 | IDで比較 |
| 不変性 | 不変 | 可変 |
| 例 | Money, Email | Order, User |

---

## 集約（Aggregate）

**関連するエンティティと値オブジェクトをグループ化し、整合性の境界を定義する**。

### 特徴

| 特徴 | 説明 |
|------|------|
| **集約ルート** | 外部からアクセスする唯一の入口 |
| **整合性の境界** | 集約内は常に整合性を保つ |
| **トランザクション境界** | 1つの集約 = 1つのトランザクション |
| **参照は ID で** | 他の集約は IDのみ で参照 |

### コード例

```typescript
// Order（集約ルート）
class Order {
  constructor(
    private readonly id: OrderId,
    private readonly customerId: CustomerId,  // 他の集約はIDで参照
    private items: OrderItem[],               // 集約内のエンティティ
    private shippingAddress: Address          // 集約内の値オブジェクト
  ) {}

  // 集約ルート経由でのみ操作
  addItem(product: ProductId, quantity: number): void {
    // 整合性チェック
    if (this.items.length >= 10) {
      throw new Error('注文は10品目までです');
    }
    this.items.push(new OrderItem(product, quantity));
  }

  removeItem(itemId: OrderItemId): void {
    this.items = this.items.filter(item => !item.id.equals(itemId));
  }
}

// OrderItem（集約内のエンティティ）
// ⚠️ 外部から直接アクセスしない！Order経由でアクセス
class OrderItem {
  constructor(
    readonly id: OrderItemId,
    readonly productId: ProductId,
    readonly quantity: number
  ) {}
}
```

### 集約設計のポイント

```
┌─────────────────────────────────────────┐
│  Order（集約ルート）                      │
│  ─────────────────────────────          │
│                                          │
│   ┌──────────────┐  ┌──────────────┐    │
│   │ OrderItem    │  │ OrderItem    │    │
│   └──────────────┘  └──────────────┘    │
│                                          │
│   ┌──────────────────────────────────┐  │
│   │ Address（値オブジェクト）          │  │
│   └──────────────────────────────────┘  │
│                                          │
└─────────────────────────────────────────┘
         ↑
    外部からはここだけ
```

---

## リポジトリ（Repository）

**集約の永続化と取得を抽象化するインターフェース**。コレクションのように振る舞う。

### 特徴

| 特徴 | 説明 |
|------|------|
| **永続化の抽象化** | SQLを隠蔽 |
| **集約ルートごと** | 1集約 = 1リポジトリ |
| **コレクション風** | `find`, `save`, `remove` |
| **インターフェース** | ドメイン層で定義、インフラ層で実装 |

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
  findById(id: OrderId): Order | null {
    const row = db.query(
      `SELECT * FROM orders WHERE id = ?`,
      id.value
    );
    return row ? Order.fromRow(row) : null;
  }

  save(order: Order): void {
    db.query(
      `INSERT INTO orders ... ON CONFLICT UPDATE ...`,
      order.toRow()
    );
  }
}
```

---

## ドメインサービス（Domain Service）

**エンティティや値オブジェクトに属さないドメインロジック**を表現する。

### いつ使う？

| 使う場面 | 例 |
|---------|---|
| 複数の集約にまたがる処理 | 在庫確認 + 注文作成 |
| 外部サービスとの連携 | 決済処理 |
| 計算ロジック | 割引計算、手数料計算 |

### コード例

```typescript
// 📦 domain/services/discount-policy.ts
class DiscountPolicy {
  calculate(order: Order, customer: Customer): Money {
    let discount = new Money(0);

    // 会員ランクによる割引
    if (customer.isGoldMember()) {
      discount = discount.add(order.total().multiply(0.1));
    }

    // 大量注文割引
    if (order.itemCount() >= 10) {
      discount = discount.add(new Money(500));
    }

    return discount;
  }
}

// 📦 domain/services/transfer-service.ts
class TransferService {
  // 複数の集約（口座）にまたがる処理
  transfer(from: Account, to: Account, amount: Money): void {
    from.withdraw(amount);
    to.deposit(amount);
  }
}
```

---

## アプリケーションサービス（Application Service）

**ユースケースを調整する薄いレイヤー**。ドメインロジックは持たない。

### 特徴

| 特徴 | 説明 |
|------|------|
| **ユースケースの調整** | 処理の流れを組み立てる |
| **トランザクション管理** | 開始・コミット・ロールバック |
| **認証・認可** | 権限チェック |
| **ドメインロジックを持たない** | 薄く保つ |

### コード例

```typescript
// 🎯 application/create-order-usecase.ts
class CreateOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private customerRepo: CustomerRepository,
    private discountPolicy: DiscountPolicy,
    private transactionManager: TransactionManager
  ) {}

  @Transactional()
  execute(input: CreateOrderInput): OrderId {
    // 1. 顧客を取得
    const customer = this.customerRepo.findById(input.customerId);
    if (!customer) throw new Error('顧客が見つかりません');

    // 2. 注文を作成（ドメインロジックはOrder内）
    const order = Order.create(customer.id, input.items);

    // 3. 割引を適用（ドメインサービス）
    const discount = this.discountPolicy.calculate(order, customer);
    order.applyDiscount(discount);

    // 4. 保存
    this.orderRepo.save(order);

    return order.id;
  }
}
```

### ドメインサービス vs アプリケーションサービス

| 観点 | ドメインサービス | アプリケーションサービス |
|:----:|:-------------:|:-------------------:|
| 層 | 📦 ドメイン層 | 🎯 アプリケーション層 |
| 責務 | ビジネスロジック | ユースケースの調整 |
| 例 | 割引計算、振込処理 | 注文作成、ユーザー登録 |

---

## ドメインイベント（Domain Event）

**ドメインで起きた重要な出来事を表すオブジェクト**。

### 特徴

| 特徴 | 説明 |
|------|------|
| **過去形で命名** | `OrderPlaced`, `PaymentCompleted` |
| **不変** | 発生した事実は変わらない |
| **タイムスタンプ** | いつ発生したか |
| **疎結合** | 発行者と購読者を分離 |

### コード例

```typescript
// イベント定義
class OrderPlacedEvent {
  readonly occurredAt: Date;

  constructor(
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    readonly totalAmount: Money
  ) {
    this.occurredAt = new Date();
  }
}

// イベント発行
class Order {
  private events: DomainEvent[] = [];

  place(): void {
    this.status = OrderStatus.PLACED;
    
    // イベントを発行
    this.events.push(new OrderPlacedEvent(
      this.id,
      this.customerId,
      this.calculateTotal()
    ));
  }

  pullEvents(): DomainEvent[] {
    const events = [...this.events];
    this.events = [];
    return events;
  }
}

// イベント購読（別のコンテキストで処理）
class OrderPlacedHandler {
  handle(event: OrderPlacedEvent): void {
    // メール送信、在庫引当、ポイント付与など
    this.emailService.sendOrderConfirmation(event.customerId);
  }
}
```

---

## ファクトリ（Factory）

**複雑なオブジェクト生成ロジックをカプセル化する**。

### いつ使う？

| 使う場面 | 例 |
|---------|---|
| 生成ロジックが複雑 | 複数の依存関係がある |
| 生成時のバリデーション | 整合性チェックが必要 |
| 集約の再構築 | DBからの復元 |

### コード例

```typescript
// ファクトリメソッド（エンティティ内）
class Order {
  static create(
    customerId: CustomerId,
    items: CreateOrderItemInput[]
  ): Order {
    if (items.length === 0) {
      throw new Error('注文には1つ以上の商品が必要です');
    }

    return new Order(
      OrderId.generate(),
      customerId,
      items.map(i => new OrderItem(i.productId, i.quantity)),
      OrderStatus.PENDING
    );
  }

  // DBからの再構築
  static fromPersistence(row: OrderRow): Order {
    return new Order(
      new OrderId(row.id),
      new CustomerId(row.customer_id),
      row.items.map(OrderItem.fromPersistence),
      OrderStatus.fromString(row.status)
    );
  }
}
```

---

## 仕様（Specification）

**ビジネスルールや条件をオブジェクトとして表現する**。

### コード例

```typescript
// 仕様パターン
interface Specification<T> {
  isSatisfiedBy(entity: T): boolean;
}

class PremiumCustomerSpec implements Specification<Customer> {
  isSatisfiedBy(customer: Customer): boolean {
    return customer.totalPurchases().getValue() >= 100000
        && customer.accountAge() >= 365;
  }
}

class ActiveOrderSpec implements Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return order.status !== OrderStatus.CANCELLED
        && order.status !== OrderStatus.DELIVERED;
  }
}

// 使用例
const premiumSpec = new PremiumCustomerSpec();
const premiumCustomers = customers.filter(c => premiumSpec.isSatisfiedBy(c));
```

---

## まとめ

### 構成要素一覧

| 構成要素 | 責務 | 層 |
|---------|------|:--:|
| **値オブジェクト** | 不変の値を表現 | 📦 |
| **エンティティ** | 識別子を持つオブジェクト | 📦 |
| **集約** | 整合性の境界 | 📦 |
| **リポジトリ** | 永続化の抽象化 | 📦 interface / 🗄️ impl |
| **ドメインサービス** | エンティティに属さないロジック | 📦 |
| **アプリケーションサービス** | ユースケースの調整 | 🎯 |
| **ドメインイベント** | ドメインで起きた出来事 | 📦 |
| **ファクトリ** | 複雑な生成ロジック | 📦 |
| **仕様** | ビジネスルールのオブジェクト化 | 📦 |

### 層の凡例

- 📦 = ドメイン層
- 🎯 = アプリケーション層
- 🗄️ = インフラ層

---

## 参考リンク

- [10_tactical_design.md](./10_tactical_design.md) - 戦術的設計の概要
- [01_ddd_introduction.md](./01_ddd_introduction.md) - DDD入門
