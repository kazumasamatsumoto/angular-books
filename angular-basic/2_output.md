Angular の`@Output()`デコレータについてまとめます。

## 基本的な使い方

`@Output()`デコレータは、子コンポーネントから親コンポーネントへイベントを発信するために使用されます。

```typescript
import { Component, Output, EventEmitter } from "@angular/core";

@Component({
  selector: "app-child",
  template: `<button (click)="onClick()">クリック</button>`,
})
export class ChildComponent {
  @Output() itemEvent = new EventEmitter<string>();

  onClick() {
    this.itemEvent.emit("子コンポーネントからのデータ");
  }
}
```

親コンポーネントでの使用例:

```html
<app-child (itemEvent)="handleEvent($event)"></app-child>
```

```typescript
@Component({
  selector: "app-parent",
  template: `<app-child (itemEvent)="handleEvent($event)"></app-child>`,
})
export class ParentComponent {
  handleEvent(data: string) {
    console.log(data); // '子コンポーネントからのデータ'
  }
}
```

## 主な特徴

1. **EventEmitter との連携**:
   `@Output()`は`EventEmitter`クラスと組み合わせて使用します。`EventEmitter`は型パラメータを取り、親コンポーネントに送信するデータの型を指定できます。

2. **別名の設定**:

   ```typescript
   @Output('aliasName') eventName = new EventEmitter<string>();
   ```

3. **複数の値を送信**:
   オブジェクトを使って複数の値を一度に送信できます。

   ```typescript
   @Output() multipleValues = new EventEmitter<{name: string, value: number}>();

   sendValues() {
     this.multipleValues.emit({name: 'テスト', value: 42});
   }
   ```

4. **イベントのバブリング**:
   イベントは上位のコンポーネント階層にバブリングしないため、必要な場合は各レベルで再発信する必要があります。

## ユースケース

1. **フォーム送信**:

   ```typescript
   @Output() formSubmit = new EventEmitter<FormData>();

   onSubmit(form: FormData) {
     this.formSubmit.emit(form);
   }
   ```

2. **ページネーション**:

   ```typescript
   @Output() pageChange = new EventEmitter<number>();

   changePage(pageNumber: number) {
     this.pageChange.emit(pageNumber);
   }
   ```

3. **コンポーネント状態変更通知**:

   ```typescript
   @Output() statusChange = new EventEmitter<'active' | 'inactive'>();

   setStatus(status: 'active' | 'inactive') {
     this.statusChange.emit(status);
   }
   ```

## Input/Output の組み合わせパターン

コンポーネント間でデータを双方向にやり取りする場合、`@Input()`と`@Output()`を組み合わせたパターンが使用されます。

```typescript
@Component({
  selector: "app-counter",
  template: `
    <p>カウント: {{ count }}</p>
    <button (click)="increment()">増加</button>
  `,
})
export class CounterComponent {
  @Input() count: number = 0;
  @Output() countChange = new EventEmitter<number>();

  increment() {
    this.count++;
    this.countChange.emit(this.count);
  }
}
```

親での双方向バインディング:

```html
<app-counter [(count)]="parentCount"></app-counter>
```

## 注意点

- イベント名と出力プロパティ名は一致させることがベストプラクティス
- 複雑なロジックの場合はサービスを介した状態管理を検討する
- `ngOnDestroy`で EventEmitter をクリーンアップするとメモリリークを防止できる

`@Output()`デコレータは Angular のコンポーネント間通信の重要な要素であり、疎結合で再利用可能なコンポーネント設計を実現します。
