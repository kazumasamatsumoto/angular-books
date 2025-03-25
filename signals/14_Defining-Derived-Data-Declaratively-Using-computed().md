# Defining Derived Data Declaratively Using computed()

Angular の signal API における `computed()` 関数は、他の signal 値から派生したデータを宣言的に定義するための強力な機能です。以下にその要点をまとめます：

## computed() の基本概念

- `computed()` は他の signal から派生した値を計算するために使用します
- 元となる signal が変更されると、computed signal も自動的に更新されます
- 読み取り専用であり、直接更新することはできません

## 主な特徴

1. **メモ化（キャッシング）** - 同じ入力に対して計算を繰り返すことを避け、パフォーマンスを向上させます
2. **自動依存関係追跡** - 使用される signal を自動的に検出して監視します
3. **遅延評価** - 値が必要になるまで計算を延期します

## 使用例

```typescript
import { signal, computed } from "@angular/core";

// 基本的な signal の定義
const firstName = signal("John");
const lastName = signal("Doe");

// computed signal の作成
const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName()); // 出力: "John Doe"

// 依存する signal を更新すると、computed signal も更新される
firstName.set("Jane");
console.log(fullName()); // 出力: "Jane Doe"
```

## 複雑な計算への応用

```typescript
const items = signal([
  { name: "Item 1", price: 10, quantity: 2 },
  { name: "Item 2", price: 15, quantity: 1 },
]);

// 合計金額を計算する computed signal
const totalPrice = computed(() => {
  return items().reduce((sum, item) => sum + item.price * item.quantity, 0);
});

console.log(totalPrice()); // 出力: 35
```

## ベストプラクティス

1. 計算ロジックはピュアな関数にする（副作用なし）
2. 複雑な計算は小さな computed signal に分割する
3. 依存関係は computed 関数内で明示的に呼び出す
4. 不必要な再計算を避けるため、signal の変更を最小限に抑える

## computed() vs 通常の関数

- 通常の関数は毎回実行されますが、computed signal は依存する値が変わった場合のみ再計算されます
- computed signal は Angular の変更検出システムと統合され、効率的なレンダリングを可能にします

computed() は、複雑なアプリケーション状態を効率的に管理し、宣言的なプログラミングスタイルを促進する強力なツールです。
