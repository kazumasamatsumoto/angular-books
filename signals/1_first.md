# Angular Signal の初めての実装方法

Angular のシグナルは、リアクティブな値を作成し管理するための新しい API です。これを使うことで、コンポーネント間のデータフローをより明示的に制御できるようになります。

## 基本的なシグナルの作成

```typescript
import { signal } from "@angular/core";

// 基本的なシグナルの作成
const count = signal(0);

// 値の取得
console.log(count()); // 0

// 値の設定
count.set(1);
console.log(count()); // 1

// 現在の値に基づいて更新
count.update((value) => value + 1);
console.log(count()); // 2
```

## コンポーネントでの使用例

```typescript
import { Component, signal } from "@angular/core";

@Component({
  selector: "app-counter",
  template: `
    <p>カウンター: {{ counter() }}</p>
    <button (click)="increment()">増加</button>
    <button (click)="decrement()">減少</button>
  `,
})
export class CounterComponent {
  counter = signal(0);

  increment() {
    this.counter.update((value) => value + 1);
  }

  decrement() {
    this.counter.update((value) => value - 1);
  }
}
```

## 計算値（computed）の作成

他のシグナルに基づいて動的に値を計算する場合：

```typescript
import { signal, computed } from "@angular/core";

const price = signal(100);
const quantity = signal(2);

// 計算値（computed signal）の作成
const total = computed(() => price() * quantity());

console.log(total()); // 200

// 依存するシグナルを変更すると、計算値も自動的に更新される
price.set(150);
console.log(total()); // 300
```

## effect を使った副作用の処理

シグナルの値が変わったときに副作用を実行する方法：

```typescript
import { signal, effect } from "@angular/core";

const message = signal("Hello");

// effectを作成
effect(() => {
  console.log(`メッセージが更新されました: ${message()}`);
});

// 値を変更すると、effectが自動的に実行される
message.set("Hello Angular Signals");
// コンソールに「メッセージが更新されました: Hello Angular Signals」と表示される
```

## シグナルの特徴

- **明示的なリアクティビティ**: 変更を追跡したい値だけをシグナルとして定義
- **細かい粒度の更新**: 特定のシグナルが変更された場合のみ再計算が行われる
- **デバッグのしやすさ**: データの流れが明示的で追跡しやすい
- **パフォーマンスの向上**: 不必要な再レンダリングを減らせる

Angular のシグナルは、特に複雑なアプリケーションでデータフローを管理するための強力なツールです。RxJS と併用することもでき、用途に応じて最適なアプローチを選択できます。
