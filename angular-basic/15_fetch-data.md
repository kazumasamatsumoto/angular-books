# Angular Custom Service - Fetching Data

Angular でデータ取得を行うカスタムサービスは、外部 API やリソースとの通信を効率的に管理するための重要なコンポーネントです。以下に主要な要素とベストプラクティスをまとめます。

## 基本的なデータ取得サービス

```typescript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";
import { catchError, map, tap } from "rxjs/operators";

@Injectable({
  providedIn: "root",
})
export class DataService {
  private apiUrl = "https://api.example.com/data";

  constructor(private http: HttpClient) {}

  getData(): Observable<any[]> {
    return this.http.get<any[]>(this.apiUrl).pipe(
      tap((data) => console.log("Fetched data", data)),
      catchError(this.handleError("getData", []))
    );
  }

  private handleError<T>(operation = "operation", result?: T) {
    return (error: any): Observable<T> => {
      console.error(`${operation} failed: ${error.message}`);
      // エラーをログに記録し、アプリケーションを継続させる
      return of(result as T);
    };
  }
}
```

## 主な特徴

1. **HttpClient の使用**:

   - Angular の HttpClient モジュールは RESTful API との通信に最適化されています
   - JSON データの自動変換をサポートしています

2. **Observable の返却**:

   - サービスメソッドは通常 Observable を返します
   - これにより非同期操作を簡単に扱えます

3. **エラーハンドリング**:

   - catchError 演算子を使用して HTTP リクエストのエラーをキャッチします
   - エラーログを記録し、アプリケーションの実行を継続させます

4. **RxJS オペレータの活用**:
   - tap: サイドエフェクト（ログ記録など）を追加
   - map: レスポンスデータの変換
   - catchError: エラー処理

## コンポーネントでの使用例

```typescript
import { Component, OnInit } from "@angular/core";
import { DataService } from "./data.service";

@Component({
  selector: "app-data-list",
  template: `
    <div *ngIf="loading">Loading...</div>
    <ul *ngIf="!loading">
      <li *ngFor="let item of items">{{ item.name }}</li>
    </ul>
    <div *ngIf="error">{{ error }}</div>
  `,
})
export class DataListComponent implements OnInit {
  items: any[] = [];
  loading = false;
  error: string | null = null;

  constructor(private dataService: DataService) {}

  ngOnInit(): void {
    this.fetchData();
  }

  fetchData(): void {
    this.loading = true;
    this.dataService.getData().subscribe({
      next: (data) => {
        this.items = data;
        this.loading = false;
      },
      error: (err) => {
        this.error = "データの取得に失敗しました";
        this.loading = false;
      },
      complete: () => {
        this.loading = false;
      },
    });
  }
}
```

## 高度なデータ取得テクニック

1. **ステート管理**:
   - BehaviorSubject を使用してデータの状態を管理できます
   - これによりサービス内でデータをキャッシュし、複数のコンポーネント間で共有できます

```typescript
import { BehaviorSubject } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class DataStateService {
  private dataSubject = new BehaviorSubject<any[]>([]);
  public data$ = this.dataSubject.asObservable();

  updateData(data: any[]): void {
    this.dataSubject.next(data);
  }
}
```

2. **インターセプターの使用**:
   - HTTP インターセプターを使用して認証トークンの追加やエラー処理の一元化が可能です

```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const authToken = localStorage.getItem("token");
    const authReq = req.clone({
      headers: req.headers.set("Authorization", `Bearer ${authToken}`),
    });
    return next.handle(authReq);
  }
}
```

3. **リトライロジック**:
   - 一時的なネットワークエラーに対処するためのリトライロジックを実装できます

```typescript
getData(): Observable<any[]> {
  return this.http.get<any[]>(this.apiUrl)
    .pipe(
      retry(3), // 最大3回リトライ
      catchError(this.handleError('getData', []))
    );
}
```

Angular のカスタムサービスを使用したデータ取得は、コンポーネントからデータアクセスロジックを分離し、アプリケーションの保守性とテスト容易性を高める効果的な方法です。
