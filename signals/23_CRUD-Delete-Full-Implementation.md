# CRUD Delete - Full Implementation

CRUD における「Delete」操作の完全な実装についてまとめます。

## 1. サービスレイヤーの実装

```typescript
import { Injectable } from "@angular/core";
import { HttpClient, HttpErrorResponse } from "@angular/common/http";
import { Observable, catchError, throwError } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class CourseService {
  private apiUrl = "api/courses";

  constructor(private http: HttpClient) {}

  // Observable を返す実装
  deleteCourse(id: number): Observable<void> {
    const url = `${this.apiUrl}/${id}`;
    return this.http.delete<void>(url).pipe(catchError(this.handleError));
  }

  // async/await 用の実装
  async deleteCourseAsync(id: number): Promise<boolean> {
    const url = `${this.apiUrl}/${id}`;
    try {
      await firstValueFrom(this.http.delete<void>(url));
      return true;
    } catch (error) {
      this.handleError(error as HttpErrorResponse);
      return false;
    }
  }

  private handleError(error: HttpErrorResponse) {
    console.error("API Error:", error);
    return throwError(
      () => new Error("Failed to delete course. Please try again.")
    );
  }
}
```

## 2. コンポーネントでの実装

```typescript
import { Component, signal, inject } from "@angular/core";
import { Course } from "../models/course.model";
import { CourseService } from "../services/course.service";

@Component({
  selector: "app-course-list",
  templateUrl: "./course-list.component.html",
})
export class CourseListComponent {
  private courseService = inject(CourseService);

  courses = signal<Course[]>([]);
  isLoading = signal(false);
  message = signal<{ type: "success" | "error"; text: string } | null>(null);
  courseToDelete = signal<Course | null>(null);

  constructor() {
    this.loadCourses();
  }

  async loadCourses() {
    this.isLoading.set(true);
    try {
      const courses = await this.courseService.getCourses();
      this.courses.set(courses);
    } catch (error) {
      console.error("Failed to load courses", error);
    } finally {
      this.isLoading.set(false);
    }
  }

  // 削除の確認ダイアログを表示
  confirmDelete(course: Course) {
    this.courseToDelete.set(course);
  }

  // 削除をキャンセル
  cancelDelete() {
    this.courseToDelete.set(null);
  }

  // 削除を実行
  async deleteCourse() {
    const course = this.courseToDelete();
    if (!course) return;

    this.isLoading.set(true);

    try {
      const success = await this.courseService.deleteCourseAsync(course.id);

      if (success) {
        // UIからコースを削除
        this.courses.update((courses) =>
          courses.filter((c) => c.id !== course.id)
        );

        // 成功メッセージを表示
        this.message.set({
          type: "success",
          text: `Course "${course.title}" has been deleted successfully`,
        });

        // 数秒後にメッセージを消す
        setTimeout(() => this.message.set(null), 3000);
      } else {
        throw new Error("Failed to delete course");
      }
    } catch (error) {
      console.error("Delete error:", error);
      this.message.set({
        type: "error",
        text: "Failed to delete course. Please try again.",
      });
    } finally {
      this.isLoading.set(false);
      this.courseToDelete.set(null); // 確認ダイアログを閉じる
    }
  }
}
```

## 3. 確認ダイアログのテンプレート

```html
<!-- 確認ダイアログ -->
<div class="modal-overlay" *ngIf="courseToDelete()">
  <div class="modal-content">
    <h3>Confirm Deletion</h3>
    <p>
      Are you sure you want to delete the course "{{ courseToDelete()?.title
      }}"?
    </p>
    <p class="warning">This action cannot be undone!</p>

    <div class="modal-actions">
      <button class="btn-secondary" (click)="cancelDelete()">Cancel</button>
      <button class="btn-danger" (click)="deleteCourse()">Delete</button>
    </div>
  </div>
</div>
```

## 4. コース一覧のテンプレート

```html
<div class="courses-container">
  <!-- ローディングインジケーター -->
  @if (isLoading()) {
  <div class="loading-spinner">Loading...</div>
  }

  <!-- メッセージ表示 -->
  @if (message()) {
  <div
    class="alert"
    [ngClass]="message()?.type === 'success' ? 'alert-success' : 'alert-error'"
  >
    {{ message()?.text }}
  </div>
  }

  <!-- コース一覧 -->
  <div class="course-list">
    @for (course of courses(); track course.id) {
    <div class="course-card">
      <h3>{{ course.title }}</h3>
      <p>{{ course.description }}</p>
      <div class="course-details">
        <span>Category: {{ course.category }}</span>
        <span>Level: {{ course.level }}</span>
      </div>
      <div class="actions">
        <button class="btn-edit" (click)="editCourse(course)">Edit</button>
        <button class="btn-delete" (click)="confirmDelete(course)">
          Delete
        </button>
      </div>
    </div>
    } @empty {
    <div class="empty-state">
      <p>No courses available</p>
    </div>
    }
  </div>
</div>
```

## 5. 削除機能のスタイリング

```css
/* 確認ダイアログ用のスタイル */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  padding: 24px;
  border-radius: 8px;
  width: 400px;
  max-width: 90%;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.warning {
  color: #d9534f;
  font-weight: bold;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  margin-top: 24px;
}

/* ボタンスタイル */
.btn-secondary {
  padding: 8px 16px;
  background-color: #6c757d;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-danger {
  padding: 8px 16px;
  background-color: #d9534f;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

/* アラートメッセージ */
.alert {
  padding: 12px;
  margin-bottom: 16px;
  border-radius: 4px;
}

.alert-success {
  background-color: #d4edda;
  color: #155724;
}

.alert-error {
  background-color: #f8d7da;
  color: #721c24;
}
```

## 6. アニメーションを追加した削除実装

```typescript
import { Component, signal, inject } from "@angular/core";
import { trigger, transition, style, animate } from "@angular/animations";

@Component({
  selector: "app-course-list",
  templateUrl: "./course-list.component.html",
  animations: [
    trigger("itemAnimation", [
      transition(":leave", [
        style({ opacity: 1, height: "*" }),
        animate("300ms ease-out", style({ opacity: 0, height: 0, margin: 0 })),
      ]),
    ]),
  ],
})
export class CourseListComponent {
  // 前述の実装と同じ...
}
```

対応する HTML の変更:

```html
<div class="course-list">
  @for (course of courses(); track course.id) {
  <div class="course-card" [@itemAnimation]>
    <!-- 内容は前述と同じ -->
  </div>
  }
</div>
```

## 7. 一括削除機能の実装

```typescript
selectedCourses = signal<Set<number>>(new Set());

toggleSelection(courseId: number) {
  this.selectedCourses.update(selected => {
    const newSelection = new Set(selected);
    if (newSelection.has(courseId)) {
      newSelection.delete(courseId);
    } else {
      newSelection.add(courseId);
    }
    return newSelection;
  });
}

isSelected(courseId: number): boolean {
  return this.selectedCourses().has(courseId);
}

selectAll() {
  const allIds = this.courses().map(course => course.id);
  this.selectedCourses.set(new Set(allIds));
}

clearSelection() {
  this.selectedCourses.set(new Set());
}

async deleteSelected() {
  const selectedIds = Array.from(this.selectedCourses());
  if (selectedIds.length === 0) return;

  this.isLoading.set(true);

  try {
    // 並行して削除リクエストを実行
    const results = await Promise.allSettled(
      selectedIds.map(id => this.courseService.deleteCourseAsync(id))
    );

    // 成功した削除をカウント
    const successCount = results.filter(
      result => result.status === 'fulfilled' && result.value === true
    ).length;

    // UIを更新
    this.courses.update(courses =>
      courses.filter(course => !this.selectedCourses().has(course.id))
    );

    // 結果メッセージを表示
    this.message.set({
      type: 'success',
      text: `Successfully deleted ${successCount} out of ${selectedIds.length} courses`
    });

    // 選択をクリア
    this.clearSelection();

  } catch (error) {
    console.error('Error during bulk delete:', error);
    this.message.set({
      type: 'error',
      text: 'An error occurred during bulk delete'
    });
  } finally {
    this.isLoading.set(false);
  }
}
```

## 実装のポイント

1. **削除前の確認**

   - 誤操作防止のため確認ダイアログを表示
   - 取り消せない操作であることを明示

2. **エラーハンドリング**

   - サーバーからのエラーレスポンスを適切に処理
   - わかりやすいエラーメッセージの表示

3. **ユーザーフィードバック**

   - 操作の結果を明確に表示
   - ローディング状態の表示

4. **UI 更新**

   - 削除後のリスト表示の更新
   - アニメーションによる視覚的フィードバック

5. **複数選択と一括削除**
   - 効率的な操作のための追加機能
   - 部分的な成功/失敗の処理

適切な削除機能の実装により、ユーザーは安全かつ効率的にデータを管理することができます。確認プロセス、視覚的フィードバック、エラー処理を組み合わせることで、ユーザー体験の質を高めることができます。
