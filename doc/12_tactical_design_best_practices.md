# 戦術的設計のベストプラクティス

戦術的設計を実践する際の推奨パターンとアンチパターンをまとめます。

## 目次
1. [値オブジェクトのベストプラクティス](#値オブジェクトのベストプラクティス)
2. [エンティティのベストプラクティス](#エンティティのベストプラクティス)
3. [集約のベストプラクティス](#集約のベストプラクティス)
4. [リポジトリのベストプラクティス](#リポジトリのベストプラクティス)
5. [アプリケーションサービスのベストプラクティス](#アプリケーションサービスのベストプラクティス)
6. [一般的なアンチパターン](#一般的なアンチパターン)
7. [まとめ](#まとめ)

---

## 値オブジェクトのベストプラクティス

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

### ✅ 推奨パターン

| パターン | 説明 |
|---------|------|
| **IDで同一性を判断** | 属性が変わってもIDが同じなら同一 |
| **状態変更はメソッド経由** | setterを公開しない |
| **不変条件を守る** | 常に有効な状態を保つ |
| **ファクトリメソッドで生成** | `Order.create()` でバリデーション |

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
