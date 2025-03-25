# Loading Data With OnInit or afterNextRender

## 概要

Angular でデータ読み込みを行うタイミングとして、従来の`ngOnInit`と新しい`afterNextRender`の 2 つの主要なアプローチがあります。それぞれに適した用途と特徴があります。

## ngOnInit によるデータ読み込み

### 特徴

- コンポーネントのライフサイクルフックを使用
- コンポーネントの初期化時に一度だけ実行される
- 従来の Angular アプリケーションの標準的なアプローチ

### 実装例

```typescript
@Component({...})
export class ProductListComponent implements OnInit {
  constructor(private productService: ProductService) {}

  ngOnInit() {
    this.productService.fetchProducts();
  }
}
```

### メリット

- シンプルで明確なライフサイクル
- コンポーネント初期化のための標準メカニズム
- サーバーサイドレンダリング（SSR）との互換性が高い

### デメリット

- レンダリング前にデータ取得が開始されるため、初期表示が遅くなる可能性
- 複数の子コンポーネントがある場合、親が表示される前に多くのデータ取得が発生する可能性

## afterNextRender によるデータ読み込み

### 特徴

- Angular 16 以降で導入された新しいアプローチ
- コンポーネントが実際にレンダリングされた後にデータを読み込む
- クライアントサイドでの UX を重視したアプローチ

### 実装例

```typescript
@Component({...})
export class ProductListComponent {
  constructor(
    private productService: ProductService,
    private renderer: Renderer2
  ) {
    afterNextRender(() => {
      this.productService.fetchProducts();
    }, { phase: AfterRenderPhase.Write });
  }
}
```

### メリット

- 初期レンダリングが高速化（先に UI が表示される）
- ユーザーエクスペリエンスの向上
- リソースの段階的な読み込みが可能
- レンダリングの優先順位付けが容易

### デメリット

- SSR との統合が複雑になる可能性
- データがない初期状態の UI ハンドリングが必要
- 相対的に新しい機能のため、ベストプラクティスがまだ発展中

## 選択ガイドライン

### ngOnInit を選ぶべき場合

- SSR を使用するアプリケーション
- SEO 最適化が重要なアプリケーション
- データなしでは UI が意味をなさない場合
- 従来の Angular パターンを継続したい場合

### afterNextRender を選ぶべき場合

- クライアントサイドのパフォーマンス最適化が重要
- データがなくてもある程度機能する UI がある場合
- 段階的なコンテンツ読み込みを実装したい場合
- スケルトン UI やプレースホルダーを使用する場合

## 両方のアプローチの併用

多くの現代的な Angular アプリケーションでは、コンポーネントの重要度や性質に応じて両方のアプローチを組み合わせています：

- クリティカルなデータは`ngOnInit`で読み込む
- 補助的なデータは`afterNextRender`で読み込む

これにより、初期表示の速度とデータの完全性のバランスを取ることができます。

適切なアプローチの選択は、アプリケーションの要件、ターゲットユーザー、パフォーマンス目標によって異なります。
