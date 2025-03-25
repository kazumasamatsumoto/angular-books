# CRUD Read Operation - Defining The API of our Service Layer

## サービスレイヤー API の定義要点

1. **データモデル定義**:

   - 明確なインターフェースまたはクラスでエンティティ構造を定義
   - 型安全性を確保するため TypeScript の利点を活用

2. **サービスインターフェース設計**:

   - 読み取り操作のための明確なメソッド定義
   - 同期/非同期アクセスパターンの適切な選択

3. **Signals の活用**:

   - データストリームを Signal として公開
   - `signal()`、`computed()`の適切な使用

4. **エラーハンドリング戦略**:

   - 読み取り操作失敗時の適切なエラー処理メカニズム
   - エラー状態を Signal で伝播

5. **HTTP 通信設計**:
   - HttpClient モジュールとの統合
   - エンドポイント URL の設計と管理

## コード例の概要

```typescript
// データモデル
interface Item {
  id: number;
  name: string;
  description: string;
}

// サービスレイヤー
@Injectable({
  providedIn: "root",
})
export class ItemService {
  // Signalによる状態管理
  private itemsSignal = signal<Item[]>([]);
  private loadingSignal = signal<boolean>(false);
  private errorSignal = signal<string | null>(null);

  // 読み取り専用のパブリックAPI
  public items = this.itemsSignal.asReadonly();
  public loading = this.loadingSignal.asReadonly();
  public error = this.errorSignal.asReadonly();

  constructor(private http: HttpClient) {}

  // データ取得メソッド
  fetchItems(): void {
    this.loadingSignal.set(true);
    this.errorSignal.set(null);

    this.http
      .get<Item[]>("/api/items")
      .pipe(finalize(() => this.loadingSignal.set(false)))
      .subscribe({
        next: (data) => this.itemsSignal.set(data),
        error: (err) => this.errorSignal.set(err.message),
      });
  }

  // 単一アイテム取得
  getItemById(id: number): Signal<Item | undefined> {
    return computed(() => this.items().find((item) => item.id === id));
  }
}
```

## 設計原則

- **関心の分離**: データアクセスロジックを UI から分離
- **単一責任**: サービスは特定のデータエンティティの操作に集中
- **カプセル化**: 内部実装の詳細を隠し、クリーンな API を提供
- **イミュータビリティ**: データ変更は Signal.set()を通じて行い、直接変更しない
- **リアクティビティ**: データの変更が UI に自動的に反映される仕組みを提供

この設計アプローチにより、クリーンなアーキテクチャと Signals を活用したモダンな Angular アプリケーションの基盤を構築できます。
