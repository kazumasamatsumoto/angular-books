# Angular Custom Service - Data Modification with an HTTP PUT

Angular で HTTP PUT リクエストを使用してデータを更新するカスタムサービスについて説明します。PUT メソッドは既存のリソースを完全に置き換えるために使用され、RESTful API との通信において重要な役割を果たします。

## 基本的な HTTP PUT サービス

```typescript
import { Injectable } from "@angular/core";
import { HttpClient, HttpHeaders } from "@angular/common/http";
import { Observable, of } from "rxjs";
import { catchError, tap } from "rxjs/operators";
import { Item } from "./item.model";

@Injectable({
  providedIn: "root",
})
export class ItemService {
  private apiUrl = "https://api.example.com/items";

  httpOptions = {
    headers: new HttpHeaders({ "Content-Type": "application/json" }),
  };

  constructor(private http: HttpClient) {}

  // PUT: 既存のアイテムを更新する
  updateItem(item: Item): Observable<Item> {
    const url = `${this.apiUrl}/${item.id}`;
    return this.http.put<Item>(url, item, this.httpOptions).pipe(
      tap((_) => console.log(`updated item id=${item.id}`)),
      catchError(this.handleError<Item>("updateItem"))
    );
  }

  private handleError<T>(operation = "operation", result?: T) {
    return (error: any): Observable<T> => {
      console.error(`${operation} failed: ${error.message}`);
      // エラーをログに記録し、空の結果を返してアプリケーションを継続させる
      return of(result as T);
    };
  }
}
```

## PUT リクエストの主な特徴

1. **完全な置き換え**:

   - PUT は対象リソースを完全に置き換えるため、すべての必要なフィールドを送信する必要があります
   - 部分的な更新には PATCH メソッドが適しています

2. **URL パラメータ**:

   - 通常、更新するリソースの ID を URL に含めます: `/api/items/{id}`

3. **リクエストヘッダー**:

   - Content-Type ヘッダーを 'application/json' に設定して JSON データを送信します

4. **レスポンス処理**:
   - 更新されたリソースが返されることが一般的です
   - エラーケースも適切に処理する必要があります

## コンポーネントでの使用例

```typescript
import { Component, OnInit } from "@angular/core";
import { ItemService } from "./item.service";
import { Item } from "./item.model";

@Component({
  selector: "app-item-edit",
  template: `
    <div *ngIf="item">
      <h2>編集: {{ item.name }}</h2>
      <div>
        <label>ID: {{ item.id }}</label>
      </div>
      <div>
        <label
          >名前:
          <input [(ngModel)]="item.name" placeholder="名前" />
        </label>
      </div>
      <div>
        <label
          >説明:
          <input [(ngModel)]="item.description" placeholder="説明" />
        </label>
      </div>
      <button (click)="save()">保存</button>
      <div *ngIf="error">{{ error }}</div>
      <div *ngIf="success">更新が完了しました</div>
    </div>
  `,
})
export class ItemEditComponent implements OnInit {
  item: Item | undefined;
  error: string | null = null;
  success = false;

  constructor(private itemService: ItemService) {}

  ngOnInit(): void {
    // 別のメソッドでアイテムを取得すると仮定
    this.getItem(1);
  }

  getItem(id: number): void {
    this.itemService.getItem(id).subscribe((item) => (this.item = item));
  }

  save(): void {
    if (this.item) {
      this.itemService.updateItem(this.item).subscribe({
        next: () => {
          this.success = true;
          this.error = null;
        },
        error: (err) => {
          this.error = "更新に失敗しました。";
          this.success = false;
        },
      });
    }
  }
}
```

## 応用例と最適化テクニック

1. **楽観的ロック機構**:
   - バージョンフィールドや最終更新日時を使用して競合を検出

```typescript
updateItem(item: Item): Observable<Item> {
  const url = `${this.apiUrl}/${item.id}`;
  // If-Match ヘッダーを使用してバージョンを確認
  const headers = new HttpHeaders({
    'Content-Type': 'application/json',
    'If-Match': `"${item.version}"`
  });

  return this.http.put<Item>(url, item, { headers })
    .pipe(
      tap(_ => console.log(`updated item id=${item.id}`)),
      catchError(error => {
        if (error.status === 412) { // Precondition Failed
          return throwError(() => new Error('このアイテムは他のユーザーによって更新されました。'));
        }
        return throwError(() => error);
      })
    );
}
```

2. **フォームとの連携**:
   - Reactive Forms を使用した更新処理

```typescript
// コンポーネント内
this.itemForm = this.fb.group({
  id: [item.id],
  name: [item.name, Validators.required],
  description: [item.description]
});

save(): void {
  if (this.itemForm.valid) {
    const updatedItem = { ...this.item, ...this.itemForm.value };
    this.itemService.updateItem(updatedItem).subscribe(/* ... */);
  }
}
```

3. **バッチ更新処理**:
   - 複数アイテムの一括更新

```typescript
updateItems(items: Item[]): Observable<Item[]> {
  // forkJoin で複数の PUT リクエストを並行実行
  const updateObservables = items.map(item =>
    this.http.put<Item>(`${this.apiUrl}/${item.id}`, item, this.httpOptions)
  );

  return forkJoin(updateObservables).pipe(
    catchError(this.handleError<Item[]>('updateItems', []))
  );
}
```

HTTP PUT を利用したデータ更新サービスは、Angular アプリケーションにおいて効率的なデータ管理を実現するための重要なコンポーネントです。適切なエラー処理と最適化テクニックを組み合わせることで、堅牢なデータ更新機能を実装できます。
