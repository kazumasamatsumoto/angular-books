# Skipping The Loading Indicator Using an HTTP Context

## 概要

HTTP Context を使用することで、特定の HTTP リクエストでローディングインジケーターを表示せずにスキップする機能を実装できます。これは、バックグラウンドで自動的に実行される頻繁なリクエストや、ユーザーに表示する必要のないリクエストに特に有用です。

## HTTP Context とは

HTTP Context は、Angular 12 以降で導入された HTTP クライアントの機能で、リクエスト単位でメタデータを提供できます。これにより、インターセプターやその他の HTTP パイプラインコンポーネントに追加情報を渡すことが可能になります。

## 主要コンポーネント

1. **HTTP Context トークンの定義**

   - ローディングをスキップするための専用トークン
   - インターセプターでの判定に使用

2. **インターセプターでの Context 確認**

   - リクエストから Context を抽出
   - ローディング表示のスキップ判定

3. **リクエスト実行時の Context 設定**
   - 特定のリクエストに Context を追加
   - サービスレベルでの簡易 API 提供

## 実装例

### HTTP Context トークンの定義

```typescript
// loading-skip.token.ts
import { HttpContextToken } from "@angular/common/http";

// ローディングをスキップするためのトークン（デフォルトはfalse）
export const SKIP_LOADING_INDICATOR = new HttpContextToken(() => false);
```

### ローディングインターセプターの拡張

```typescript
// loading.interceptor.ts
import { Injectable, inject } from "@angular/core";
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpContext,
} from "@angular/common/http";
import { Observable, finalize } from "rxjs";
import { SharedLoadingService } from "./shared-loading.service";
import { SKIP_LOADING_INDICATOR } from "./loading-skip.token";

@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  private loadingService = inject(SharedLoadingService);

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // Contextからスキップフラグを取得
    const skipLoading = request.context.get(SKIP_LOADING_INDICATOR);

    // スキップする場合は、ローディングを表示せずにリクエストを実行
    if (skipLoading) {
      return next.handle(request);
    }

    // 通常のローディング処理
    const operationId = this.generateOperationId(request);
    const description = this.getRequestDescription(request);

    this.loadingService.startLoading(operationId, description);

    return next.handle(request).pipe(
      finalize(() => {
        this.loadingService.endLoading(operationId);
      })
    );
  }

  private generateOperationId(request: HttpRequest<any>): string {
    return `http-${request.method}-${request.url}-${Date.now()}`;
  }

  private getRequestDescription(request: HttpRequest<any>): string {
    // 説明生成ロジック（省略）
    return `リクエスト処理中: ${request.method} ${request.url}`;
  }
}
```

### HTTP サービスでの使用例

```typescript
// data.service.ts
import { Injectable, inject } from "@angular/core";
import { HttpClient, HttpContext } from "@angular/common/http";
import { Observable } from "rxjs";
import { SKIP_LOADING_INDICATOR } from "./loading-skip.token";

@Injectable({
  providedIn: "root",
})
export class DataService {
  private http = inject(HttpClient);

  // 通常のリクエスト（ローディングインジケーターを表示）
  fetchCourses(): Observable<any[]> {
    return this.http.get<any[]>("/api/courses");
  }

  // バックグラウンドリクエスト（ローディングインジケーターをスキップ）
  fetchUserPreferences(): Observable<any> {
    // Contextを使用してローディングをスキップ
    const context = new HttpContext().set(SKIP_LOADING_INDICATOR, true);

    return this.http.get<any>("/api/user/preferences", { context });
  }

  // 自動更新のリクエスト（ローディングインジケーターをスキップ）
  autoSaveProgress(courseId: string, progress: any): Observable<any> {
    const context = new HttpContext().set(SKIP_LOADING_INDICATOR, true);

    return this.http.post(`/api/courses/${courseId}/progress`, progress, {
      context,
    });
  }
}
```

### 便利なヘルパーメソッドの作成

```typescript
// http-helpers.service.ts
import { Injectable } from "@angular/core";
import { HttpClient, HttpContext } from "@angular/common/http";
import { Observable } from "rxjs";
import { SKIP_LOADING_INDICATOR } from "./loading-skip.token";

@Injectable({
  providedIn: "root",
})
export class HttpHelpersService {
  constructor(private http: HttpClient) {}

  // ローディングインジケーターを表示せずにGETリクエストを実行
  getWithoutLoading<T>(url: string, options: any = {}): Observable<T> {
    const context = new HttpContext().set(SKIP_LOADING_INDICATOR, true);
    return this.http.get<T>(url, { ...options, context });
  }

  // ローディングインジケーターを表示せずにPOSTリクエストを実行
  postWithoutLoading<T>(
    url: string,
    body: any,
    options: any = {}
  ): Observable<T> {
    const context = new HttpContext().set(SKIP_LOADING_INDICATOR, true);
    return this.http.post<T>(url, body, { ...options, context });
  }

  // その他のHTTPメソッド...
}
```

## 使用シナリオ

### 自動保存機能

ユーザーの入力中に自動的にデータを保存する機能：

```typescript
// auto-save.component.ts
import { Component, inject, signal, effect } from "@angular/core";
import { DataService } from "./data.service";
import { debounceTime, Subject } from "rxjs";

@Component({
  selector: "app-editor",
  template: `
    <div class="editor">
      <textarea
        [ngModel]="content()"
        (ngModelChange)="updateContent($event)"
      ></textarea>

      <div class="status">
        <span *ngIf="isSaved()">保存済み</span>
        <span *ngIf="!isSaved()">編集中...</span>
      </div>
    </div>
  `,
})
export class EditorComponent {
  private dataService = inject(DataService);
  private saveSubject = new Subject<string>();

  content = signal("");
  isSaved = signal(true);

  constructor() {
    // 変更から500ms後に自動保存
    this.saveSubject.pipe(debounceTime(500)).subscribe((content) => {
      this.autoSave(content);
    });
  }

  updateContent(newContent: string) {
    this.content.set(newContent);
    this.isSaved.set(false);
    this.saveSubject.next(newContent);
  }

  private autoSave(content: string) {
    // ローディングインジケーターをスキップしてバックグラウンドで保存
    this.dataService
      .autoSaveProgress("current-document", { content })
      .subscribe({
        next: () => {
          this.isSaved.set(true);
        },
        error: (err) => {
          console.error("自動保存中にエラーが発生しました:", err);
          // エラー通知（ローディングインジケーターなし）
        },
      });
  }
}
```

### ポーリングリクエスト

定期的に更新情報を取得するバックグラウンドリクエスト：

```typescript
// notifications.service.ts
import { Injectable, inject } from "@angular/core";
import { HttpClient, HttpContext } from "@angular/common/http";
import { interval, switchMap } from "rxjs";
import { SKIP_LOADING_INDICATOR } from "./loading-skip.token";

@Injectable({
  providedIn: "root",
})
export class NotificationsService {
  private http = inject(HttpClient);

  // 通知の定期チェック
  startNotificationPolling() {
    // 30秒ごとに通知を確認
    return interval(30000).pipe(switchMap(() => this.checkNewNotifications()));
  }

  private checkNewNotifications() {
    // ローディングインジケーターをスキップ
    const context = new HttpContext().set(SKIP_LOADING_INDICATOR, true);
    return this.http.get("/api/notifications/unread", { context });
  }
}
```

## 条件付きスキップ

状況に応じてローディングインジケーターの表示をスキップする：

```typescript
// data.service.ts
fetchCourses(options: { silently?: boolean } = {}): Observable<any[]> {
  let request = this.http.get<any[]>('/api/courses');

  // silentlyオプションが指定された場合はローディングをスキップ
  if (options.silently) {
    const context = new HttpContext().set(SKIP_LOADING_INDICATOR, true);
    request = this.http.get<any[]>('/api/courses', { context });
  }

  return request;
}

// 使用例
// 通常のローディング表示付きで取得
this.dataService.fetchCourses();

// サイレントに取得（ローディングインジケーターなし）
this.dataService.fetchCourses({ silently: true });
```

## メリット

1. **柔軟性の向上** - リクエスト単位でローディング動作をカスタマイズ可能
2. **ユーザーエクスペリエンスの改善** - 不要なローディングインジケーターの表示を防止
3. **バックグラウンド処理の最適化** - ユーザーの作業を妨げずに処理を実行
4. **既存コードへの影響の最小化** - 選択的に適用でき、既存のコードを変更する必要性が低い

## 注意点

1. **一貫性の確保** - どのリクエストでローディングをスキップするかの明確な基準を設定
2. **デバッグの複雑さ** - スキップされるリクエストの追跡が難しくなる可能性
3. **エラー処理** - スキップされたリクエストのエラー通知方法の検討

HTTP Context を使用したローディングインジケーターのスキップは、より洗練されたユーザーエクスペリエンスを提供するための優れた方法です。バックグラウンド処理、自動保存、ポーリングなどのシナリオで特に有用です。
