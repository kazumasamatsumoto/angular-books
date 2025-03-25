# Angular HTTP クライアント - リクエストパラメータを使用した GET 呼び出し

Angular の HTTP クライアントは、外部 API やサーバーとの通信を行うための強力なモジュールです。最新バージョンの Angular では、HTTP 通信に関して重要な変更点がいくつか導入されています。

## 基本的な HTTP クライアントの使用方法

### 1. HttpClientModule のインポート (Angular 15 以前)

```typescript
// app.module.ts
import { HttpClientModule } from "@angular/common/http";

@NgModule({
  imports: [HttpClientModule],
  // ...
})
export class AppModule {}
```

### スタンドアロンコンポーネントでの使用 (Angular 16+)

Angular 16 以降では、provideHttpClient() 関数を使用します：

```typescript
// main.ts
import { bootstrapApplication } from "@angular/platform-browser";
import { provideHttpClient } from "@angular/common/http";
import { AppComponent } from "./app/app.component";

bootstrapApplication(AppComponent, {
  providers: [provideHttpClient()],
}).catch((err) => console.error(err));
```

## HTTP GET リクエストの実行

### 基本的な GET リクエスト

```typescript
import { HttpClient } from "@angular/common/http";
import { Injectable } from "@angular/core";
import { Observable } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class DataService {
  private apiUrl = "https://api.example.com/data";

  constructor(private http: HttpClient) {}

  getData(): Observable<any> {
    return this.http.get(this.apiUrl);
  }
}
```

### クエリパラメータを使用した GET リクエスト

```typescript
import { HttpClient, HttpParams } from '@angular/common/http';

// ...

getDataWithParams(page: number, limit: number): Observable<any> {
  // パラメータの構築
  const params = new HttpParams()
    .set('page', page.toString())
    .set('limit', limit.toString());

  // パラメータを含むリクエスト
  return this.http.get(this.apiUrl, { params });
}
```

### 複数の値を持つパラメータ

```typescript
// 複数の値を持つパラメータ (例: ?category=books&category=electronics)
getProductsByCategories(categories: string[]): Observable<any> {
  let params = new HttpParams();

  categories.forEach(category => {
    params = params.append('category', category);
  });

  return this.http.get(this.apiUrl, { params });
}
```

## Angular 17+ の新機能: 型安全な HTTP クライアント

Angular 17 以降では、型安全性が強化された HTTP クライアントが導入されました：

```typescript
// 型安全な HTTP リクエスト
interface Product {
  id: number;
  name: string;
  price: number;
}

getProducts(): Observable<Product[]> {
  return this.http.get<Product[]>(this.apiUrl);
}

// withJsonpSupport() などの追加機能もサポート
```

## HTTP リクエストオプション

```typescript
getDataWithOptions(): Observable<any> {
  return this.http.get(this.apiUrl, {
    params: new HttpParams().set('category', 'books'),
    headers: new HttpHeaders({
      'Authorization': 'Bearer token123',
      'Content-Type': 'application/json'
    }),
    observe: 'response', // 完全なレスポンス（ヘッダー含む）を取得
    responseType: 'json'  // レスポンスの形式を指定
  });
}
```

## HTTP インターセプター

インターセプターを使用すると、すべての HTTP リクエストを傍受して変更できます。例えば、認証トークンの追加やエラー処理などに使用します：

```typescript
// auth.interceptor.ts
import { Injectable } from "@angular/core";
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
} from "@angular/common/http";
import { Observable } from "rxjs";

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(
    request: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    // トークンを追加したリクエストを作成
    const authToken = this.authService.getToken();
    const authReq = request.clone({
      headers: request.headers.set("Authorization", `Bearer ${authToken}`),
    });

    // 変更したリクエストで続行
    return next.handle(authReq);
  }
}
```

### インターセプターの提供 (Angular 15 以前)

```typescript
// app.module.ts
@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
  ]
})
```

### インターセプターの提供 (Angular 16+)

```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [provideHttpClient(withInterceptors([authInterceptor]))],
});
```

## エラーハンドリング

```typescript
import { catchError, throwError } from 'rxjs';

getData(): Observable<any> {
  return this.http.get(this.apiUrl).pipe(
    catchError(error => {
      console.error('Error fetching data:', error);
      return throwError(() => new Error('Something went wrong; please try again later.'));
    })
  );
}
```

## Angular 18 での変更点 (プレビュー)

Angular 18 では、以下の新機能が予定されています：

1. **HTTP コンテキスト**: リクエスト固有のメタデータを渡すためのコンテキスト機能
2. **withRequestsMadeViaParent()**: 親インジェクターを通じてリクエストを行うオプション
3. **レスポンスタイプの改善**: より厳密な型安全性

```typescript
// Angular 18 プレビュー例
const context = new HttpContext().set(RETRY_COUNT, 3);

this.http.get("/api/data", { context }).subscribe(/* ... */);
```

## ベストプラクティス

1. **厳格な型付け**: `http.get<User[]>()` のように常に型を指定する
2. **環境変数の使用**: API URL は `environment.ts` ファイルで管理する
3. **サービス抽象化**: HTTP 呼び出しはサービス内にカプセル化する
4. **キャッシュ戦略**: 頻繁に変更されないデータはキャッシュを検討する
5. **リトライロジック**: 不安定なネットワーク接続に対しては rxjs の `retry()` を使用する

```typescript
import { retry, catchError } from 'rxjs/operators';

getData(): Observable<any> {
  return this.http.get(this.apiUrl).pipe(
    retry(3), // 3回リトライ
    catchError(this.handleError)
  );
}
```

Angular の HTTP クライアントは、型安全でリアクティブなアプローチで API 通信を実装する強力なツールです。最新バージョンではより洗練された機能が追加され、開発体験が向上しています。
