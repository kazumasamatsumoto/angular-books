# Signal-Based Loading Indicator - Shared Signal Service

## 概要

Shared Signal Service は、Angular Signals を活用して複数のコンポーネント間でローディング状態を共有する仕組みです。これにより、アプリケーション全体でローディング状態を統一的に管理でき、ユーザーエクスペリエンスが向上します。

## 主要コンポーネント

1. **中央集権的な状態管理**

   - アプリケーション全体のローディング状態を単一のサービスで管理
   - 複数のコンポーネントやサービスからアクセス可能

2. **複数のローディング状態の追跡**

   - 異なる操作や機能ごとにユニークなローディング状態を管理
   - 複数の同時ローディング処理の統合

3. **Signal API の活用**
   - `signal()`, `computed()`, `effect()`を使った状態管理
   - 効率的な変更検知と更新伝播

## 実装例

### SharedLoadingService

```typescript
// shared-loading.service.ts
import { Injectable, signal, computed, effect } from "@angular/core";

interface LoadingOperation {
  id: string;
  description: string;
  isActive: boolean;
  startTime: number;
}

@Injectable({
  providedIn: "root",
})
export class SharedLoadingService {
  // プライベートSignals
  private activeOperations = signal<LoadingOperation[]>([]);

  // パブリックcomputed Signals
  public isLoading = computed(() => this.activeOperations().length > 0);
  public currentOperations = computed(() => this.activeOperations());
  public operationCount = computed(() => this.activeOperations().length);

  // 最も長く実行されている操作
  public longestRunningOperation = computed(() => {
    const operations = this.activeOperations();
    if (operations.length === 0) return null;

    return operations.reduce((prev, current) =>
      prev.startTime < current.startTime ? prev : current
    );
  });

  constructor() {
    // デバッグ用のeffect
    effect(() => {
      if (this.isLoading()) {
        console.debug(`ローディング中: ${this.operationCount()}件の処理`);
      }
    });
  }

  startLoading(id: string, description: string = "ローディング中"): void {
    this.activeOperations.update((ops) => [
      ...ops.filter((op) => op.id !== id),
      {
        id,
        description,
        isActive: true,
        startTime: Date.now(),
      },
    ]);
  }

  endLoading(id: string): void {
    this.activeOperations.update((ops) => ops.filter((op) => op.id !== id));
  }

  // 特定の操作グループに関連するローディング状態を取得
  getGroupLoading(groupPrefix: string) {
    return computed(() => {
      return this.activeOperations().some((op) =>
        op.id.startsWith(groupPrefix)
      );
    });
  }

  // すべてのローディング操作をクリア
  clearAll(): void {
    this.activeOperations.set([]);
  }
}
```

### アプリケーションレベルのインジケーターコンポーネント

```typescript
// app-loading-indicator.component.ts
import { Component, inject } from "@angular/core";
import { SharedLoadingService } from "./shared-loading.service";

@Component({
  selector: "app-loading-indicator",
  template: `
    <div class="global-loading-overlay" *ngIf="loadingService.isLoading()">
      <div class="loading-spinner"></div>
      <div class="loading-details">
        <p>{{ loadingService.operationCount() }}件の処理実行中</p>
        @if (loadingService.longestRunningOperation(); as op) {
        <small
          >{{ op.description }} ({{
            getElapsedTime(op.startTime)
          }}秒経過)</small
        >
        }
      </div>
    </div>
  `,
  styles: [
    /* スタイル省略 */
  ],
})
export class AppLoadingIndicatorComponent {
  loadingService = inject(SharedLoadingService);

  getElapsedTime(startTime: number): number {
    return Math.round((Date.now() - startTime) / 1000);
  }
}
```

### コンポーネント固有のローディングインジケーター

```typescript
// component-loading.component.ts
import { Component, Input, inject } from "@angular/core";
import { SharedLoadingService } from "./shared-loading.service";

@Component({
  selector: "app-component-loading",
  template: `
    <div class="component-loading" *ngIf="isComponentLoading()">
      <div class="spinner-small"></div>
      <span>{{ loadingText }}</span>
    </div>
  `,
  styles: [
    /* スタイル省略 */
  ],
})
export class ComponentLoadingComponent {
  @Input() groupPrefix: string = "";
  @Input() loadingText: string = "ロード中...";

  private loadingService = inject(SharedLoadingService);

  // グループに関連するローディング状態を取得
  isComponentLoading = computed(() => {
    if (!this.groupPrefix) return this.loadingService.isLoading();
    return this.loadingService.getGroupLoading(this.groupPrefix)();
  });
}
```

## 使用例

### HTTP サービスとの統合

```typescript
// data.service.ts
import { Injectable, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable, finalize } from "rxjs";
import { SharedLoadingService } from "./shared-loading.service";

@Injectable({
  providedIn: "root",
})
export class DataService {
  private http = inject(HttpClient);
  private loadingService = inject(SharedLoadingService);

  fetchCourses(): Observable<any[]> {
    const operationId = "fetch-courses";
    this.loadingService.startLoading(operationId, "コース一覧を取得中");

    return this.http
      .get<any[]>("/api/courses")
      .pipe(finalize(() => this.loadingService.endLoading(operationId)));
  }

  saveUserProgress(courseId: string, progress: any): Observable<any> {
    const operationId = `save-progress-${courseId}`;
    this.loadingService.startLoading(operationId, "進捗を保存中");

    return this.http
      .post(`/api/courses/${courseId}/progress`, progress)
      .pipe(finalize(() => this.loadingService.endLoading(operationId)));
  }
}
```

### コンポーネントでの使用

```typescript
// course-list.component.ts
import { Component, inject, signal } from "@angular/core";
import { DataService } from "./data.service";

@Component({
  selector: "app-course-list",
  template: `
    <div class="course-container">
      <h2>コース一覧</h2>

      <!-- コンポーネント固有のローディングインジケーター -->
      <app-component-loading
        groupPrefix="fetch-courses"
        loadingText="コースを読み込み中..."
      >
      </app-component-loading>

      <div class="course-grid">
        @for (course of courses(); track course.id) {
        <div class="course-card">
          <h3>{{ course.title }}</h3>
          <p>{{ course.description }}</p>
          <button (click)="markComplete(course.id)">
            完了としてマーク
            <!-- インライン進捗保存ローダー -->
            <app-component-loading
              [groupPrefix]="'save-progress-' + course.id"
              loadingText="保存中"
            >
            </app-component-loading>
          </button>
        </div>
        } @empty {
        <p>コースが見つかりません</p>
        }
      </div>
    </div>
  `,
})
export class CourseListComponent {
  dataService = inject(DataService);
  courses = signal<any[]>([]);

  ngOnInit() {
    this.loadCourses();
  }

  loadCourses() {
    this.dataService.fetchCourses().subscribe((data) => {
      this.courses.set(data);
    });
  }

  markComplete(courseId: string) {
    const progress = { completed: true, completedAt: new Date() };
    this.dataService.saveUserProgress(courseId, progress).subscribe({
      next: () => {
        // 成功処理
      },
      error: (err) => {
        console.error("保存エラー:", err);
      },
    });
  }
}
```

## 高度な機能と拡張

1. **タイムアウト検出**
   - 長時間実行されている処理の検出と通知

```typescript
// 長時間実行されている操作を監視
effect(() => {
  const longRunning = this.activeOperations().find(
    (op) => Date.now() - op.startTime > 10000 // 10秒以上
  );

  if (longRunning) {
    console.warn(
      `操作 ${longRunning.id} が長時間実行されています: ${Math.round(
        (Date.now() - longRunning.startTime) / 1000
      )}秒`
    );
  }
});
```

2. **優先度ベースのローディング表示**
   - 重要度に応じたローディング表示の差別化

```typescript
// 重要度を追加
interface LoadingOperation {
  id: string;
  description: string;
  isActive: boolean;
  startTime: number;
  priority: 'high' | 'medium' | 'low';
}

// 重要度の高い操作があるかを確認
public hasHighPriorityOperation = computed(() => {
  return this.activeOperations().some(op => op.priority === 'high');
});
```

3. **ローディング履歴の追跡**
   - デバッグ目的で過去のローディング操作を記録

## メリット

1. **一貫したユーザーエクスペリエンス** - アプリケーション全体で統一されたローディング表示
2. **コード再利用性** - 同じローディングロジックを複数の場所で活用
3. **デバッグのしやすさ** - 中央集権的な管理により問題特定が容易
4. **メンテナンス性** - ローディングロジックの変更が一箇所で済む
5. **柔軟性** - 単一の操作から複雑な操作グループまで対応可能

## 注意点

1. **パフォーマンス考慮** - 大量のローディング操作がある場合の Signal 更新頻度に注意
2. **クリーンアップ** - 完了しなかったローディング操作を確実に終了させる仕組みの実装
3. **視覚的一貫性** - 異なるコンポーネントでのローディング表示の一貫性を保つ

Shared Signal Service を使用することで、Angular アプリケーション全体でローディング状態を効果的に管理し、ユーザーに一貫した視覚的フィードバックを提供できます。
