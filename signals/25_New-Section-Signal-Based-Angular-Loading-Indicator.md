# Signal-Based Angular Loading Indicator

Angular Signals を活用したローディングインジケーターの実装についてまとめます。

## 概要

Signals を使用したローディングインジケーターは、アプリケーション全体または特定のコンポーネントの非同期処理状態を効率的に管理し、ユーザーに視覚的フィードバックを提供します。

## 主要な機能

1. **グローバル/ローカルロード状態の管理**

   - アプリケーション全体または個別のコンポーネントの読み込み状態を追跡
   - 複数の非同期処理を一元管理

2. **シグナルベースの状態管理**

   - 読み込み状態をリアクティブに追跡
   - 複数の状態（開始、進行中、完了、エラー）をシームレスに管理

3. **UI との統合**
   - 異なるローディング状態に応じたインジケーター表示
   - アニメーションやトランジションとの連携

## 実装例

```typescript
// loading.service.ts
import { Injectable, signal, computed } from "@angular/core";

@Injectable({
  providedIn: "root",
})
export class LoadingService {
  // メイン状態シグナル
  private loadingOperations = signal<Map<string, boolean>>(new Map());

  // 派生した計算値
  public isLoading = computed(() => {
    return Array.from(this.loadingOperations()).some(([_, value]) => value);
  });

  // 現在のローディング操作数
  public operationCount = computed(() => {
    return Array.from(this.loadingOperations()).filter(([_, value]) => value)
      .length;
  });

  // 新しいローディング操作を開始
  startLoading(operationId: string): void {
    this.loadingOperations.update((map) => {
      const newMap = new Map(map);
      newMap.set(operationId, true);
      return newMap;
    });
  }

  // ローディング操作を完了
  endLoading(operationId: string): void {
    this.loadingOperations.update((map) => {
      const newMap = new Map(map);
      newMap.set(operationId, false);
      return newMap;
    });
  }
}
```

```typescript
// loading-indicator.component.ts
import { Component, inject } from "@angular/core";
import { LoadingService } from "./loading.service";

@Component({
  selector: "app-loading-indicator",
  template: `
    <div class="loading-container" *ngIf="loadingService.isLoading()">
      <div class="spinner"></div>
      <div class="loading-text">
        読み込み中... ({{ loadingService.operationCount() }}件の処理)
      </div>
    </div>
  `,
  styles: [
    `
      .loading-container {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background-color: rgba(0, 0, 0, 0.3);
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        z-index: 9999;
      }

      .spinner {
        width: 50px;
        height: 50px;
        border: 5px solid #f3f3f3;
        border-top: 5px solid #3498db;
        border-radius: 50%;
        animation: spin 1s linear infinite;
      }

      @keyframes spin {
        0% {
          transform: rotate(0deg);
        }
        100% {
          transform: rotate(360deg);
        }
      }
    `,
  ],
})
export class LoadingIndicatorComponent {
  loadingService = inject(LoadingService);
}
```

## 使用例

```typescript
// データを取得するコンポーネント
import { Component, OnInit, inject } from "@angular/core";
import { DataService } from "./data.service";
import { LoadingService } from "./loading.service";

@Component({
  selector: "app-data-component",
  template: `<div>データ: {{ data() | json }}</div>`,
})
export class DataComponent implements OnInit {
  private dataService = inject(DataService);
  private loadingService = inject(LoadingService);

  data = signal<any[]>([]);

  ngOnInit() {
    this.fetchData();
  }

  fetchData() {
    const operationId = "fetch-data-" + Date.now();
    this.loadingService.startLoading(operationId);

    this.dataService.getData().subscribe({
      next: (result) => {
        this.data.set(result);
        this.loadingService.endLoading(operationId);
      },
      error: (err) => {
        console.error("データ取得エラー:", err);
        this.loadingService.endLoading(operationId);
      },
    });
  }
}
```

## HTTP インターセプターとの統合

```typescript
// loading.interceptor.ts
import { Injectable } from "@angular/core";
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
} from "@angular/common/http";
import { Observable, finalize } from "rxjs";
import { LoadingService } from "./loading.service";

@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  constructor(private loadingService: LoadingService) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const operationId = `http-${request.method}-${request.url}-${Date.now()}`;
    this.loadingService.startLoading(operationId);

    return next.handle(request).pipe(
      finalize(() => {
        this.loadingService.endLoading(operationId);
      })
    );
  }
}
```

## 高度な機能

1. **進行状況インジケーター**

   - ファイルアップロードやダウンロードの進捗状況を表示

2. **コンテキスト別インジケーター**

   - 特定の UI 領域に紐づけられたローダー

3. **スケルトンローディング**

   - コンテンツの形状を表すプレースホルダーの表示

4. **遅延ローディングインジケーター**
   - 短時間の処理では表示せず、長時間の処理のみインジケーターを表示

## メリット

1. **リアクティブ性** - 状態変化に即座に反応
2. **コード整理** - 関心事の分離が明確
3. **拡張性** - 様々なローディングパターンに対応可能
4. **デバッグしやすさ** - ローディング状態の追跡が容易

Signals を活用することで、従来の RxJS ベースのアプローチよりもシンプルかつ直感的なローディングインジケーターの実装が可能になります。
