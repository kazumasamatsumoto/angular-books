# Angular のビルトインパイプ - 完全カタログ

Angular のビルトインパイプはテンプレート内でデータ変換を簡単に行うための強力なツールです。以下に主要なビルトインパイプとその使用方法をまとめます。

## 文字列変換パイプ

### 1. UpperCasePipe

文字列を大文字に変換します。

```html
<p>{{ 'hello' | uppercase }}</p>
<!-- 出力: HELLO -->
```

### 2. LowerCasePipe

文字列を小文字に変換します。

```html
<p>{{ 'HELLO' | lowercase }}</p>
<!-- 出力: hello -->
```

### 3. TitleCasePipe

各単語の先頭を大文字に変換します。

```html
<p>{{ 'hello world' | titlecase }}</p>
<!-- 出力: Hello World -->
```

## 数値変換パイプ

### 4. DecimalPipe

数値をフォーマットします。

```html
<p>{{ 3.14159 | number:'1.2-2' }}</p>
<!-- 出力: 3.14 -->
```

### 5. PercentPipe

数値をパーセント形式に変換します。

```html
<p>{{ 0.259 | percent:'2.2-2' }}</p>
<!-- 出力: 25.90% -->
```

### 6. CurrencyPipe

数値を通貨形式に変換します。

```html
<p>{{ 123.45 | currency:'JPY':'symbol':'1.0-0' }}</p>
<!-- 出力: ¥123 -->
```

## 日付・時刻パイプ

### 7. DatePipe

日付をフォーマットします。

```html
<p>{{ dateObj | date:'yyyy-MM-dd' }}</p>
<!-- 出力: 2025-03-25 -->
```

## 配列・オブジェクト変換パイプ

### 8. JsonPipe

オブジェクトを JSON 文字列に変換します。デバッグに便利です。

```html
<pre>{{ userObject | json }}</pre>
```

### 9. SlicePipe

配列や文字列の一部を切り出します。

```html
<p>{{ [1,2,3,4,5] | slice:1:3 }}</p>
<!-- 出力: [2,3] -->
```

### 10. KeyValuePipe

オブジェクトのキーと値のペアを配列に変換します。

```html
<div *ngFor="let item of obj | keyvalue">{{item.key}}: {{item.value}}</div>
```

## 変更検知パイプ

### 11. AsyncPipe

Observable やプロミスを購読し、最新の値を返します。自動的にサブスクリプション管理も行います。

```html
<p>{{ dataObservable | async }}</p>
```

## 条件パイプ

### 12. I18nPluralPipe

数値に基づいて複数形表現を選択します。

```html
<p>{{ messageCount | i18nPlural:messageMapping }}</p>
<!-- messageMapping = {'=0': '0 messages', '=1': '1 message', 'other': '# messages'} -->
```

### 13. I18nSelectPipe

値に基づいて文字列を選択します。

```html
<p>{{ gender | i18nSelect:genderMapping }}</p>
<!-- genderMapping = {'male': '彼', 'female': '彼女', 'other': '他'} -->
```

## パイプのパラメータと連鎖

パイプはコロン(`:`)を使ってパラメータを受け取ることができます。

```html
{{ value | pipe:param1:param2 }}
```

また、パイプは連鎖させることも可能です。

```html
{{ value | pipe1 | pipe2 | pipe3 }}
```

## カスタムパイプ

ビルトインパイプだけでなく、独自のカスタムパイプも作成できます。

```typescript
@Pipe({
  name: "myCustomPipe",
})
export class MyCustomPipe implements PipeTransform {
  transform(value: any, ...args: any[]): any {
    // 変換ロジック
    return transformedValue;
  }
}
```

## 純粋と不純なパイプ

- **純粋なパイプ**: 入力値が変わった場合のみ再実行（デフォルト）
- **不純なパイプ**: 変更検知のたびに再実行 (`pure: false` オプションを使用)

アプリケーションのパフォーマンスを考慮して、可能な限り純粋なパイプを使用することが推奨されています。

## パイプのベストプラクティス

1. 単純なデータ変換のためにパイプを使用する
2. 複雑なビジネスロジックはコンポーネントやサービスに移動させる
3. パフォーマンスが重要な場合、不純なパイプの使用を最小限に抑える
4. 必要に応じて、複数のパイプを連鎖させて使用する

Angular のビルトインパイプを活用することで、テンプレート内のデータ表示を効率的に制御できます。
