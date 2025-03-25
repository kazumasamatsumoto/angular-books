# Fetching Data From the Service Layer With Error Handling

## 概要

サービスレイヤーからデータを取得する際の効果的なエラーハンドリングは、堅牢な Angular アプリケーション開発において重要な側面です。以下にその主要なポイントをまとめます。

## 主要コンポーネント

1. **サービスレイヤーの設計**

   - エラー状態を管理するための Signals の使用
   - ローディング状態の追跡
   - エラーメッセージの一元管理

2. **エラーハンドリングの実装方法**

   - catchError オペレーターの活用
   - エラータイプの適切な分類
   - ユーザーフレンドリーなエラーメッセージへの変換

3. **コンポーネントでのエラー表示**
   - エラー状態に基づく条件付きレンダリング
   - ユーザーへの適切なフィードバック提供
   - リトライメカニズムの提供

## 実装例

```typescript
// サービスレイヤー
@Injectable({
  providedIn: "root",
})
export class DataService {
  // 状態管理用Signals
  private itemsSignal = signal<Item[]>([]);
  private loadingSignal = signal<boolean>(false);
  private errorSignal = signal<string | null>(null);

  // 公開API
  public items = this.itemsSignal.asReadonly();
  public loading = this.loadingSignal.asReadonly();
  public error = this.errorSignal.asReadonly();

  constructor(private http: HttpClient) {}

  fetchItems(): void {
    // ローディング状態をセット
    this.loadingSignal.set(true);
    this.errorSignal.set(null);

    this.http
      .get<Item[]>("/api/items")
      .pipe(
        catchError((error: HttpErrorResponse) => {
          // エラータイプに基づいた適切なエラーメッセージ
          let errorMsg: string;

          if (error.status === 0) {
            errorMsg =
              "ネットワーク接続に問題があります。インターネット接続を確認してください。";
          } else if (error.status === 404) {
            errorMsg = "リクエストされたリソースが見つかりませんでした。";
          } else if (error.status === 500) {
            errorMsg =
              "サーバーエラーが発生しました。後ほど再試行してください。";
          } else {
            errorMsg =
              error.message || "データ取得中に予期せぬエラーが発生しました。";
          }

          // エラー状態をセット
          this.errorSignal.set(errorMsg);
          return throwError(() => new Error(errorMsg));
        }),
        finalize(() => this.loadingSignal.set(false))
      )
      .subscribe({
        next: (data) => this.itemsSignal.set(data),
        error: () => {
          // エラーは既にcatchErrorで処理済み
        },
      });
  }
}

// コンポーネント側での使用
@Component({
  selector: "app-items",
  template: `
    @if (dataService.loading()) {
    <div class="loading-spinner">読み込み中...</div>
    } @else if (dataService.error()) {
    <div class="error-container">
      <p class="error-message">{{ dataService.error() }}</p>
      <button (click)="retry()">再試行</button>
    </div>
    } @else if (dataService.items().length) {
    <ul class="items-list">
      @for (item of dataService.items(); track item.id) {
      <li>{{ item.name }}</li>
      }
    </ul>
    } @else {
    <p>データがありません</p>
    }
  `,
})
export class ItemsComponent {
  constructor(public dataService: DataService) {}

  ngOnInit() {
    this.fetchData();
  }

  fetchData() {
    this.dataService.fetchItems();
  }

  retry() {
    this.fetchData();
  }
}
```

## ベストプラクティス

1. **エラーの分類**

   - ネットワークエラー、サーバーエラー、認証エラーなど異なるタイプのエラーを区別
   - エラータイプに応じた適切な対応を提供

2. **グレースフルデグラデーション**

   - エラー発生時でも UI の一部機能は保持
   - ユーザーの操作を妨げない

3. **リトライメカニズム**

   - 一時的なエラーに対するリトライ機能の提供
   - 指数バックオフなどの高度なリトライ戦略

4. **グローバルエラーハンドリング**

   - 共通のエラーハンドリングサービスを実装
   - HTTP_INTERCEPTOR を使用した集中型エラー処理

5. **ユーザーフィードバック**
   - ローディング状態の明示
   - エラーメッセージは技術的詳細を避け、ユーザーに理解しやすい表現を使用
   - 可能な解決策の提示

Signals を活用したこのアプローチにより、エラー状態の管理が簡素化され、UI とデータ取得ロジックの間の一貫した連携が実現します。
