Angular の input デコレータについてまとめます。

`@Input()`デコレータは、Angular コンポーネントで親コンポーネントからデータを受け取るために使用される重要な機能です。

## 基本的な使い方

```typescript
import { Component, Input } from "@angular/core";

@Component({
  selector: "app-child",
  template: `<p>{{ item }}</p>`,
})
export class ChildComponent {
  @Input() item: string;
}
```

親コンポーネントからの使用例:

```html
<app-child [item]="parentData"></app-child>
```

## 主な特徴

1. **別名の設定**:
   ```typescript
   @Input('aliasName') propertyName: string;
   ```
2. **required モディファイア** (Angular 14 以降):

   ```typescript
   @Input({ required: true }) item!: string;
   ```

3. **デフォルト値の設定**:

   ```typescript
   @Input() item: string = 'デフォルト値';
   ```

4. **transform 機能** (Angular 16 以降):
   ```typescript
   @Input({
     transform: (value: string) => value.toUpperCase()
   }) item: string;
   ```

## OnChanges ライフサイクルフックとの連携

```typescript
import { Component, Input, OnChanges, SimpleChanges } from "@angular/core";

@Component({
  selector: "app-child",
  template: `<p>{{ item }}</p>`,
})
export class ChildComponent implements OnChanges {
  @Input() item: string;

  ngOnChanges(changes: SimpleChanges) {
    if (changes["item"]) {
      console.log("前の値:", changes["item"].previousValue);
      console.log("現在の値:", changes["item"].currentValue);
      console.log("初回変更?:", changes["item"].firstChange);
    }
  }
}
```

## インターセプター (Angular 17)

Angular 17 以降では、set/get アクセサーを使って入力プロパティの変更を監視・制御できます：

```typescript
private _item: string;

@Input()
set item(value: string) {
  this._item = value;
  // 値が変更されたときの追加処理
}

get item(): string {
  return this._item;
}
```

## 注意点

- 単方向データバインディングのため、子コンポーネントでの変更は親に反映されません
- オブジェクトや配列は参照渡しのため、内部の変更は親コンポーネントに影響します
- パフォーマンス向上のためには OnPushChangeDetectionStrategy と組み合わせると効果的です

これが Angular の@Input デコレータの主な特徴と使用方法です。
