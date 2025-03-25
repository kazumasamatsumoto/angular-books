# Angular HTTP Service Layers - Two Different Options

Angular における HTTP サービスレイヤーの実装には、主に 2 つの異なるアプローチがあります。

## オプション 1: HTTP クライアントの直接使用

**特徴:**

- コンポーネントが直接 HttpClient を注入して使用
- シンプルで導入が容易
- 小規模アプリケーションや単純なユースケースに適している

**実装例:**

```typescript
@Component({...})
export class ProductComponent {
  products: Product[] = [];

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.http.get<Product[]>('/api/products')
      .subscribe(data => this.products = data);
  }
}
```

**利点:**

- 少ないコード量で実装できる
- 学習コストが低い
- プロトタイピングが迅速

**欠点:**

- ビジネスロジックと HTTP ロジックが混在
- テストが困難
- コードの再利用性が低い
- エラーハンドリングの重複

## オプション 2: 専用サービスレイヤー

**特徴:**

- HTTP 通信を専用のサービスクラスにカプセル化
- リポジトリパターンに類似
- 大規模アプリケーションや複雑なデータフローに適している

**実装例:**

```typescript
@Injectable({providedIn: 'root'})
export class ProductService {
  constructor(private http: HttpClient) {}

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products')
      .pipe(
        catchError(this.handleError),
        shareReplay(1)
      );
  }

  private handleError(error: HttpErrorResponse) {
    // エラーハンドリングロジック
  }
}

@Component({...})
export class ProductComponent {
  products: Product[] = [];

  constructor(private productService: ProductService) {}

  ngOnInit() {
    this.productService.getProducts()
      .subscribe(data => this.products = data);
  }
}
```

**利点:**

- 関心の分離
- テスト容易性の向上
- コードの再利用
- キャッシュ戦略の実装が容易
- 一貫したエラーハンドリング
- API エンドポイントの一元管理

**欠点:**

- 初期設定に時間がかかる
- ボイラープレートコードが増える
- 小規模プロジェクトではオーバーエンジニアリングになる可能性

## 選択ガイドライン

**オプション 1 を選ぶ場合:**

- プロトタイプやデモプロジェクト
- 単一コンポーネントでのみ使用するデータ
- 非常に小規模なプロジェクト

**オプション 2 を選ぶ場合:**

- チーム開発プロジェクト
- 複数のコンポーネントで共有されるデータ
- 単体テストが重要なプロジェクト
- 将来の拡張が予想されるプロジェクト
- 複雑なエラーハンドリングや認証が必要

最近の Angular ベストプラクティスでは、特に Signals と組み合わせる場合、オプション 2 の専用サービスレイヤーアプローチが推奨されています。このアプローチにより、データ取得とビジネスロジックの分離、クリーンなアーキテクチャの実現が可能になります。
