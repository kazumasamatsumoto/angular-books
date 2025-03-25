# Angular Service Layers With The HTTP Client and async / await

## 概要

Angular の HttpClient と async/await パターンを組み合わせたサービスレイヤーの実装について解説します。この組み合わせにより、より読みやすく管理しやすい非同期コードを作成できます。

## 主要コンポーネント

### 1. HttpClient の基本

- HttpClient は Angular の標準 API で RESTful 操作を提供
- Observable ベースで設計されているが、Promise/async-await でも使用可能
- Injectable サービスを通じてアプリケーション全体で再利用可能

### 2. 従来の Observable アプローチから async/await への移行

**従来のアプローチ (Observable):**

```typescript
@Injectable({ providedIn: "root" })
export class ProductService {
  constructor(private http: HttpClient) {}

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>("/api/products").pipe(
      catchError((error) => {
        console.error("Error fetching products", error);
        return of([]);
      })
    );
  }
}

// コンポーネントでの使用
this.productService.getProducts().subscribe((products) => {
  this.products = products;
});
```

**async/await アプローチ:**

```typescript
@Injectable({providedIn: 'root'})
export class ProductService {
  constructor(private http: HttpClient) {}

  async getProducts(): Promise<Product[]> {
    try {
      // lastValueFrom でObservableをPromiseに変換
      return await lastValueFrom(this.http.get<Product[]>('/api/products'));
    } catch (error) {
      console.error('Error fetching products', error);
      return [];
    }
  }
}

// コンポーネントでの使用
async loadProducts() {
  this.products = await this.productService.getProducts();
}
```

### 3. Signals と async/await の統合

```typescript
@Injectable({ providedIn: "root" })
export class ProductService {
  private productsSignal = signal<Product[]>([]);
  private loadingSignal = signal<boolean>(false);
  private errorSignal = signal<string | null>(null);

  // 公開API
  public products = this.productsSignal.asReadonly();
  public loading = this.loadingSignal.asReadonly();
  public error = this.errorSignal.asReadonly();

  constructor(private http: HttpClient) {}

  async fetchProducts(): Promise<void> {
    try {
      this.loadingSignal.set(true);
      this.errorSignal.set(null);

      const products = await lastValueFrom(
        this.http.get<Product[]>("/api/products")
      );

      this.productsSignal.set(products);
    } catch (error) {
      this.errorSignal.set(this.handleError(error));
    } finally {
      this.loadingSignal.set(false);
    }
  }

  private handleError(error: any): string {
    // エラー処理ロジック
    return error instanceof HttpErrorResponse
      ? `サーバーエラー: ${error.status}`
      : "不明なエラーが発生しました";
  }
}
```

## 利点と欠点

### 利点

1. **読みやすさの向上**

   - try/catch 構文によるエラーハンドリングが直感的
   - コールバックネストが減少し、コードの流れが明確

2. **デバッグの容易さ**

   - 同期コードのようにステップ実行が可能
   - スタックトレースが理解しやすい

3. **トランザクション的な処理**
   - 複数の非同期操作を順番に実行する際に便利
   - 一連の操作の各ステップでエラーハンドリングが容易

### 欠点

1. **リアクティブプログラミングの制限**

   - ストリーム操作（map、filter、combineLatest 等）の利点を活かせない
   - キャンセレーションの実装が複雑になる

2. **メモリ管理への注意**
   - Observable の購読解除が自動的に行われない
   - Promise はキャンセルできないためリソースリークのリスク

## ベストプラクティス

1. **使い分けの基準**

   - 単純な HTTP リクエスト → async/await
   - 複雑なストリーム操作 → Observable

2. **エラーハンドリング**

   - グローバルとローカルのエラーハンドリングを適切に組み合わせる
   - エラーの種類に応じた対応を実装

3. **パフォーマンス考慮事項**

   - 大量の並列リクエストには Promise.all を活用
   - 適切なキャッシュ戦略を実装

4. **テスト容易性**
   - モックとスタブを活用した単体テスト
   - 非同期テストのための TestBed の適切な設定

適切な場面で async/await パターンを活用することで、Angular アプリケーションのサービスレイヤーをより管理しやすく、読みやすいコードにすることができます。
