# Angular Computed Signals - すべてを知る

Angular の計算シグナル（Computed Signals）は、Angular 16 で導入された新しいリアクティブプログラミング機能です。この機能は Angular アプリケーションの状態管理と UI 更新の方法を大きく改善しています。

## 計算シグナルとは

計算シグナル（computed signal）は、他のシグナルに基づいて値を計算する派生シグナルです。元となるシグナルが変更されると、計算シグナルも自動的に更新されます。

```typescript
import { computed, signal } from "@angular/core";

// 基本シグナルを作成
const count = signal(0);
const multiplier = signal(2);

// 計算シグナルを作成
const doubledCount = computed(() => count() * multiplier());

// 基本シグナルの値を変更すると...
count.set(5);
// 計算シグナルは自動的に更新される
console.log(doubledCount()); // 10
```

## 主な特徴

1. **自動依存関係追跡**: 計算シグナルは参照している他のシグナルを自動的に追跡し、それらが変更された時のみ再計算します。

2. **メモ化**: 同じ入力に対して同じ出力を返すため、パフォーマンスが向上します。不要な再計算を防ぎます。

3. **読み取り専用**: 計算シグナルは読み取り専用で、直接設定することはできません。

4. **遅延評価**: 値は実際に使用される時にのみ計算されます。

## 使用例

### 1. フィルタリングと変換

```typescript
const todos = signal([
  { id: 1, text: "Learn Angular", completed: false },
  { id: 2, text: "Build an app", completed: true },
]);

const completedTodos = computed(() => todos().filter((todo) => todo.completed));

const todoCount = computed(() => todos().length);
const completedCount = computed(() => completedTodos().length);
```

### 2. フォーム検証

```typescript
const username = signal("");
const password = signal("");

const isFormValid = computed(
  () => username().length >= 3 && password().length >= 6
);
```

### 3. 複数シグナルの組み合わせ

```typescript
const firstName = signal("John");
const lastName = signal("Doe");

const fullName = computed(() => `${firstName()} ${lastName()}`);
```

## ベストプラクティス

1. **純粋関数を使う**: 計算シグナルの関数は副作用を持たず、同じ入力に対して常に同じ出力を返すべきです。

2. **複雑さを分割する**: 複雑な計算は小さな計算シグナルに分割しましょう。

3. **必要な場所でのみ使用する**: すべての派生データが計算シグナルである必要はありません。単純な計算はテンプレート内で直接行うこともできます。

4. **無限ループを避ける**: 計算シグナル同士が循環参照しないように注意しましょう。

## 従来の RxJS アプローチとの比較

```typescript
// RxJSアプローチ
private count$ = new BehaviorSubject(0);
private multiplier$ = new BehaviorSubject(2);
doubledCount$ = combineLatest([this.count$, this.multiplier$]).pipe(
  map(([count, multiplier]) => count * multiplier)
);

// シグナルアプローチ
count = signal(0);
multiplier = signal(2);
doubledCount = computed(() => this.count() * this.multiplier());
```

シグナルアプローチはより直感的で、ボイラープレートコードが少なく、TypeScript との統合も優れています。

## まとめ

計算シグナルは、Angular アプリケーションにおける状態管理を簡素化し、パフォーマンスを向上させる強力な機能です。他のシグナルから派生した値を宣言的に定義でき、それらの依存関係が変更された場合にのみ再計算されるため、効率的なリアクティブプログラミングが可能になります。

Angular 16 以降でこの機能を活用することで、よりメンテナンス性が高く、効率的なアプリケーションを構築できます。
