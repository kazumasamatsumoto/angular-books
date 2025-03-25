# Angular Signal Inputs - Optional and Required Inputs

Angular のシグナル入力（Signal Inputs）は、コンポーネント間でデータを渡す新しい方法です。以下に、オプション入力と必須入力の概要をまとめます。

## 基本概念

Signal Inputs は Angular の最新機能で、従来の `@Input()` デコレータを拡張し、リアクティブなデータフローをより効率的に処理します。

## 入力シグナルの宣言方法

### 必須入力（Required Inputs）

```typescript
import { Component, input } from "@angular/core";

@Component({
  selector: "app-user-profile",
  template: `<div>User: {{ userName() }}</div>`,
})
export class UserProfileComponent {
  // 必須入力として宣言
  userName = input.required<string>();
}
```

必須入力の特徴:

- 親コンポーネントから値の提供が必須
- 値が提供されないとコンパイルエラーまたは実行時エラーが発生
- 型安全性が強化される

### オプション入力（Optional Inputs）

```typescript
import { Component, input } from "@angular/core";

@Component({
  selector: "app-greeting",
  template: `<div>Hello, {{ name() || "Guest" }}</div>`,
})
export class GreetingComponent {
  // オプション入力（デフォルト値あり）
  name = input<string>("Guest");

  // オプション入力（デフォルト値なし）
  theme = input<string>();
}
```

オプション入力の特徴:

- 親コンポーネントからの値提供は任意
- デフォルト値を設定可能
- デフォルト値がない場合は `undefined` となる

## 使用例

### 親コンポーネントでの使用

```typescript
@Component({
  selector: "app-parent",
  template: `
    <app-user-profile [userName]="currentUser" />
    <app-greeting [name]="currentUser" />
    <app-greeting />
    <!-- name はデフォルト値 'Guest' が使用される -->
  `,
})
export class ParentComponent {
  currentUser = "John Doe";
}
```

## 入力の変換とモデル化

```typescript
@Component({
  selector: "app-product",
  template: `<div>Price: {{ formattedPrice() }}</div>`,
})
export class ProductComponent {
  // 入力値を変換する例
  price = input.required<number>();

  // 入力から派生した computed signal
  formattedPrice = computed(() => `$${this.price().toFixed(2)}`);
}
```

## 主なメリット

1. **型安全性** - 必須入力による型チェックの強化
2. **明確な契約** - コンポーネントの依存関係が明示的
3. **パフォーマンスの向上** - 変更検出の最適化
4. **コード量の削減** - より簡潔な構文
5. **リアクティブな連携** - 他のシグナル API との統合が容易

## ベストプラクティス

- 必須の依存関係には `input.required()` を使用する
- オプションの入力にはデフォルト値を提供する
- 入力変換ロジックは computed シグナルを使用する
- コンポーネントのテストでは必須入力を必ず提供する

シグナル入力は、Angular アプリケーションの可読性、保守性、パフォーマンスを向上させる強力な機能です。

# Angular のシグナル入力（Signal Inputs）とは？

Angular の最新バージョンでは、コンポーネント間でデータを受け渡す新しい方法として「シグナル入力」が導入されました。従来の `@Input()` デコレータに代わる新しい方法です。

## 基本的な使い方

### 1. 従来の Input 方式

```typescript
@Component({...})
export class MyComponent {
  @Input() name: string;
  @Input() required userId: string;
}
```

### 2. 新しいシグナル Input 方式

```typescript
@Component({...})
export class MyComponent {
  // オプション入力
  name = input<string>();

  // 必須入力
  userId = input.required<string>();
}
```

## オプション入力と必須入力の違い

### オプション入力 (Optional Inputs)

- 親コンポーネントからの値提供は任意です
- 値が提供されない場合、`undefined`になります
- デフォルト値を設定することができます

```typescript
// デフォルト値なし
title = input<string>(); // 値が渡されなければundefined

// デフォルト値あり
name = input<string>("ゲスト"); // 値が渡されなければ'ゲスト'
```

### 必須入力 (Required Inputs)

- 親コンポーネントから必ず値を提供する必要があります
- 値が提供されない場合、エラーが発生します

```typescript
// 必須入力
userId = input.required<string>(); // 値が渡されなければエラー
```

## 実際の使用例

### コンポーネント定義

```typescript
@Component({
  selector: "app-user-card",
  template: `
    <div class="card">
      <h2>{{ name() }}</h2>
      <p>ID: {{ userId() }}</p>
    </div>
  `,
})
export class UserCardComponent {
  name = input<string>("名前なし"); // オプション (デフォルト値あり)
  userId = input.required<string>(); // 必須
}
```

### 親コンポーネントからの使用

```typescript
@Component({
  selector: "app-parent",
  template: `
    <!-- 正しい使用例: 必須入力を提供 -->
    <app-user-card [userId]="'123'" [name]="'田中太郎'" />

    <!-- name は省略可能、デフォルト値が使用される -->
    <app-user-card [userId]="'456'" />

    <!-- エラー: 必須の userId が提供されていない -->
    <app-user-card [name]="'山田花子'" />
  `,
})
export class ParentComponent {}
```

シグナル入力の最大の利点は、他のシグナル API と組み合わせたときに真価を発揮します。入力値が変更されると、関連するすべての計算が自動的に更新されるため、リアクティブなアプリケーションを簡単に構築できます。
