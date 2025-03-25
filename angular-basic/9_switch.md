Angular 16 で導入された`@switch`はテンプレート内での条件分岐を扱うための新しいディレクティブです。従来の`*ngIf`や`*ngSwitch`よりも簡潔で読みやすいコードを書くことができます。以下に主な特徴をまとめます。

### @switch の基本的な使い方

```typescript
@switch (condition) {
  @case (value1) {
    // value1に一致する場合の表示内容
  }
  @case (value2) {
    // value2に一致する場合の表示内容
  }
  @default {
    // どの条件にも一致しない場合の表示内容
  }
}
```

### @switch の特徴

1. **非常に読みやすい構文**: JavaScript の switch 文に似た直感的な構文です
2. **型安全**: TypeScript の型チェックが効くため、コンパイル時にエラーを検出できます
3. **パフォーマンスの向上**: 従来の`*ngIf`や`*ngSwitch`よりも効率的な処理を実現
4. **単一の要素しか表示されない**: 一度に一つの条件だけが表示されます

### 従来の\*ngSwitch との違い

```typescript
// 従来の*ngSwitch
<div [ngSwitch]="condition">
  <div *ngSwitchCase="value1">内容1</div>
  <div *ngSwitchCase="value2">内容2</div>
  <div *ngSwitchDefault>デフォルト内容</div>
</div>

// 新しい@switch
@switch (condition) {
  @case (value1) {
    <div>内容1</div>
  }
  @case (value2) {
    <div>内容2</div>
  }
  @default {
    <div>デフォルト内容</div>
  }
}
```

### 使用例

```typescript
@switch (user.role) {
  @case ('admin') {
    <admin-dashboard></admin-dashboard>
  }
  @case ('manager') {
    <manager-view></manager-view>
  }
  @default {
    <user-profile></user-profile>
  }
}
```

### 注意点

- Angular 16 以降でのみ使用可能
- スタンドアロンコンポーネントと相性が良い
- `@case`や`@default`は`@switch`の直接の子要素である必要がある

この新しい構文は Angular のテンプレート構文をより直感的で読みやすくする大きな一歩です。特に TypeScript との相性が良く、コードの保守性も向上します。

# Angular ngSwitch ディレクティブの詳細

`ngSwitch`は Angular の条件付きレンダリングを実現するコアディレクティブの一つです。JavaScript の switch 文と同様の考え方で、複数の条件分岐を簡潔に記述できます。

## 基本的な使い方

`ngSwitch`ディレクティブは 3 つの関連ディレクティブで構成されています：

1. **[ngSwitch]** - 評価する式を指定するコンテナディレクティブ
2. **\*ngSwitchCase** - 個々の条件分岐を定義
3. **\*ngSwitchDefault** - どの条件にも一致しない場合のデフォルト表示

```html
<div [ngSwitch]="expression">
  <div *ngSwitchCase="value1">値が value1 の場合に表示</div>
  <div *ngSwitchCase="value2">値が value2 の場合に表示</div>
  <div *ngSwitchDefault>どの条件にも一致しない場合に表示</div>
</div>
```

## 特徴と仕組み

- **単一一致の原則**: 一度に一つの`*ngSwitchCase`だけがレンダリングされます
- **厳密な比較**: 値の比較は`===`（厳密等価）演算子を使用します
- **構造ディレクティブ**: `*ngSwitchCase`と`*ngSwitchDefault`は構造ディレクティブなので、DOM 要素の追加・削除を制御します
- **パフォーマンス最適化**: 条件に一致しない要素は DOM から完全に削除されます

## 実装例

```typescript
import { Component } from "@angular/core";

@Component({
  selector: "app-user-display",
  template: `
    <div [ngSwitch]="userRole">
      <div *ngSwitchCase="'admin'">
        <h2>管理者ダッシュボード</h2>
        <admin-controls></admin-controls>
      </div>
      <div *ngSwitchCase="'editor'">
        <h2>編集者ツール</h2>
        <editor-tools></editor-tools>
      </div>
      <div *ngSwitchCase="'viewer'">
        <h2>閲覧者ビュー</h2>
        <content-view></content-view>
      </div>
      <div *ngSwitchDefault>
        <h2>ゲストアクセス</h2>
        <guest-view></guest-view>
      </div>
    </div>
  `,
})
export class UserDisplayComponent {
  userRole: string = "editor"; // ユーザーロールに基づいて変更
}
```

## 複雑な条件の取り扱い

同じ値に対して複数のケースを表示したい場合は、複数の要素に同じ値の`*ngSwitchCase`を設定できます：

```html
<div [ngSwitch]="accessLevel">
  <div *ngSwitchCase="'premium'">プレミアムコンテンツ</div>
  <div *ngSwitchCase="'premium'">特別オファー</div>
  <div *ngSwitchCase="'basic'">基本コンテンツ</div>
  <div *ngSwitchDefault>アクセス制限</div>
</div>
```

## ngIf との比較

- `ngIf`は単一の条件を評価するのに適しています
- `ngSwitch`は多数の相互排他的な条件を評価する場合に有効です
- 3 つ以上の条件分岐がある場合は、`ngSwitch`の方が読みやすく保守しやすいコードになります

## 注意点

- `[ngSwitch]`はプロパティバインディングなので角括弧`[]`が必要です
- `*ngSwitchCase`と`*ngSwitchDefault`は構造ディレクティブなのでアスタリスク`*`が必要です
- すべての`*ngSwitchCase`と`*ngSwitchDefault`は`[ngSwitch]`の直接の子要素である必要があります

## Angular 16 以降の@switch 構文との比較

Angular 16 で導入された新しい`@switch`構文は、より直感的で TypeScript との親和性が高いですが、`ngSwitch`はすべての Angular バージョンで利用できる安定した API です。

```html
<!-- 従来のngSwitch -->
<div [ngSwitch]="condition">
  <div *ngSwitchCase="'value1'">内容1</div>
  <div *ngSwitchCase="'value2'">内容2</div>
  <div *ngSwitchDefault>デフォルト内容</div>
</div>

<!-- 新しい@switch (Angular 16+) -->
@switch (condition) { @case ('value1') {
<div>内容1</div>
} @case ('value2') {
<div>内容2</div>
} @default {
<div>デフォルト内容</div>
} }
```

`ngSwitch`ディレクティブは、複数の条件分岐を持つ UI コンポーネントを効率的に実装するための Angular の重要なツールです。
