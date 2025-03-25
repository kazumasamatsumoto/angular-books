# Loading Indicator HTTP Interceptor

## 概要

Loading Indicator HTTP Interceptor は、Angular の HTTP リクエストを自動的に監視し、リクエスト中にローディングインジケーターを表示する仕組みです。HTTP Interceptor を使用することで、各コンポーネントでローディング状態を個別に管理する必要がなくなり、アプリケーション全体で一貫したローディング体験を提供できます。

## 主要コンポーネント

1. **HTTP Interceptor**

   - すべての HTTP リクエストを傍受
   - リクエスト開始時にローディング状態を開始
   - レスポンス受信時にローディング状態を終了

2. **Loading Service との連携**

   - Signal-based Loading Service との統合
   - リクエスト情報に基づいたローディング状態管理

3. **エラーハンドリング**
   - エラー発生時も確実にローディング状態を終了
   - オプションでエラー情報の表示

## 実装例

### HTTP Interceptor

```typescript
// loading.interceptor.ts
import { Injectable, inject } from "@angular/core";
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse,
} from "@angular/common/http";
import { Observable, finalize, catchError, throwError } from "rxjs";
import { SharedLoadingService } from "./shared-loading.service";

@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  private loadingService = inject(SharedLoadingService);

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // リクエストのURLからIDを生成
    const operationId = this.generateOperationId(request);

    // リクエストの説明を生成
    const description = this.getRequestDescription(request);

    // ローディング開始
    this.loadingService.startLoading(operationId, description);

    return next.handle(request).pipe(
      // エラーハンドリング
      catchError((error: HttpErrorResponse) => {
        // ローディング終了（エラー時）
        this.loadingService.endLoading(operationId);
        // エラーを再スロー
        return throwError(() => error);
      }),
      // 完了時のファイナライズ（成功でもエラーでも実行）
      finalize(() => {
        this.loadingService.endLoading(operationId);
      })
    );
  }

  private generateOperationId(request: HttpRequest<any>): string {
    return `http-${request.method}-${request.url}-${Date.now()}`;
  }

  private getRequestDescription(request: HttpRequest<any>): string {
    // URLからAPIエンドポイント名を抽出
    const urlParts = request.url.split("/");
    const endpoint = urlParts[urlParts.length - 1].split("?")[0];

    // HTTPメソッドに基づいた説明
    switch (request.method) {
      case "GET":
        return `データを取得中: ${endpoint}`;
      case "POST":
        return `データを送信中: ${endpoint}`;
      case "PUT":
      case "PATCH":
        return `データを更新中: ${endpoint}`;
      case "DELETE":
        return `データを削除中: ${endpoint}`;
      default:
        return `処理中: ${request.method} ${endpoint}`;
    }
  }
}
```

### アプリケーションモジュールへの登録

```typescript
// app.module.ts
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { HTTP_INTERCEPTORS, HttpClientModule } from "@angular/common/http";
import { LoadingInterceptor } from "./loading.interceptor";
import { AppComponent } from "./app.component";

@NgModule({
  declarations: [
    AppComponent,
    // その他のコンポーネント
  ],
  imports: [
    BrowserModule,
    HttpClientModule,
    // その他のモジュール
  ],
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoadingInterceptor,
      multi: true,
    },
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

### スタンドアロンコンポーネントでの登録（Angular 14+）

```typescript
// main.ts
import { bootstrapApplication } from "@angular/platform-browser";
import { AppComponent } from "./app/app.component";
import { provideHttpClient, withInterceptors } from "@angular/common/http";
import { loadingInterceptor } from "./app/loading.interceptor";

bootstrapApplication(AppComponent, {
  providers: [provideHttpClient(withInterceptors([loadingInterceptor]))],
}).catch((err) => console.error(err));
```

## 高度な機能

### フィルタリング機能

特定のリクエストをローディングインジケーターから除外する機能：

```typescript
// loading.interceptor.ts の intercept メソッド内
intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
  // 特定のリクエストを除外
  if (this.shouldSkipLoading(request)) {
    return next.handle(request);
  }

  // 通常のインターセプト処理...
}

private shouldSkipLoading(request: HttpRequest<any>): boolean {
  // 除外すべきリクエストの条件
  const skipPatterns = [
    '/api/heartbeat',
    '/api/analytics',
    '/api/logs'
  ];

  // URLパターンに一致するか確認
  return skipPatterns.some(pattern => request.url.includes(pattern));
}
```

### リクエストのグループ化

関連するリクエストをグループ化して管理：

```typescript
private generateOperationId(request: HttpRequest<any>): string {
  // リクエストのグループを特定
  let group = 'default';

  if (request.url.includes('/api/courses')) {
    group = 'courses';
  } else if (request.url.includes('/api/users')) {
    group = 'users';
  }

  return `http-${group}-${request.method}-${Date.now()}`;
}
```

### リクエスト遅延検出

長時間実行されているリクエストの検出：

```typescript
intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
  const operationId = this.generateOperationId(request);
  this.loadingService.startLoading(operationId);

  // リクエストの開始時間を記録
  const startTime = Date.now();

  // 遅延検出のタイマーを設定
  const timeoutId = setTimeout(() => {
    console.warn(`リクエストが5秒以上実行されています: ${request.url}`);
    this.loadingService.markAsDelayed(operationId);
  }, 5000);

  return next.handle(request).pipe(
    finalize(() => {
      // タイマーをクリア
      clearTimeout(timeoutId);

      // 実行時間をログ
      const duration = Date.now() - startTime;
      console.debug(`リクエスト完了: ${request.url} (${duration}ms)`);

      this.loadingService.endLoading(operationId);
    })
  );
}
```

## ベストプラクティス

1. **リクエスト識別子の一意性**

   - 各リクエストに一意の ID を割り当て
   - タイムスタンプなどを使用して衝突を防止

2. **デバウンス処理**

   - 短時間のリクエストでローディングインジケーターがちらつくことを防止

   ```typescript
   // 300ms未満のリクエストはローディングを表示しない
   startLoading(operationId: string): void {
     const timeoutId = setTimeout(() => {
       this.activeOperations.update(/* ... */);
     }, 300);

     this.timeouts.set(operationId, timeoutId);
   }

   endLoading(operationId: string): void {
     // タイムアウトをクリア
     const timeoutId = this.timeouts.get(operationId);
     if (timeoutId) {
       clearTimeout(timeoutId);
       this.timeouts.delete(operationId);
     }

     this.activeOperations.update(/* ... */);
   }
   ```

3. **キャンセル処理への対応**
   - リクエストキャンセル時のローディング状態の終了
   ```typescript
   return next.handle(request).pipe(
     takeUntil(this.cancelService.onCancel$(operationId)),
     finalize(() => {
       this.loadingService.endLoading(operationId);
     })
   );
   ```

## 実装例のユースケース

### グローバルローディングインジケーター

アプリケーション全体の HTTP リクエスト状態を表示：

```typescript
// app.component.html
<div class="app-container">
  <header>
    <!-- ヘッダーコンテンツ -->
  </header>

  <main>
    <router-outlet></router-outlet>
  </main>

  <footer>
    <!-- フッターコンテンツ -->
  </footer>

  <!-- グローバルローディングインジケーター -->
  <div class="global-loading" *ngIf="loadingService.isLoading()">
    <div class="spinner"></div>
    <p>{{ loadingService.currentOperations()[0]?.description || 'ロード中...' }}</p>
  </div>
</div>
```

### コンポーネント固有インジケーター

コンポーネント内で HTTP リクエストのグループをフィルタリングして表示：

```typescript
// course-list.component.ts
@Component({
  selector: "app-course-list",
  template: `
    <div class="course-container">
      <h2>コース一覧</h2>

      <!-- コース関連のリクエストのみを表示 -->
      <div class="component-loading" *ngIf="isCourseLoading()">
        コースデータを読み込み中...
      </div>

      <!-- コース一覧 -->
      <div class="course-list">
        <!-- コースリスト表示 -->
      </div>
    </div>
  `,
})
export class CourseListComponent {
  private loadingService = inject(SharedLoadingService);

  // コース関連のリクエストのみをフィルタリング
  isCourseLoading = computed(() => {
    return this.loadingService
      .currentOperations()
      .some((op) => op.id.includes("courses"));
  });
}
```

## メリット

1. **コードの一元化** - ローディング状態管理のロジックが一箇所に集約される
2. **DRY の原則** - 各コンポーネントでのローディングコードの重複を防止
3. **一貫性** - アプリケーション全体で一貫したローディングインジケーターの挙動
4. **メンテナンス性** - ローディングロジックの変更がアプリケーション全体に適用される
5. **デバッグのしやすさ** - すべての HTTP リクエストが一箇所で監視可能

HTTP Interceptor を使用したロード状態の管理は、特に大規模な Angular アプリケーションで効果的なアプローチであり、ユーザーエクスペリエンスの向上とコードの品質改善に役立ちます。
