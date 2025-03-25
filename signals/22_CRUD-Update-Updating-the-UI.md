# CRUD Update - Updating the UI

CRUD 操作の「Update」処理が成功した後の UI の更新方法についてまとめます。

## 基本的な UI 更新の流れ

1. API を通してデータを更新
2. 成功レスポンスを受け取る
3. UI 上のデータを更新
4. ユーザーへフィードバックを表示

## シグナルを活用した UI 更新

```typescript
import { Component, signal } from "@angular/core";
import { Course } from "../models/course.model";
import { CourseService } from "../services/course.service";

@Component({
  selector: "app-course-list",
  templateUrl: "./course-list.component.html",
})
export class CourseListComponent {
  courses = signal<Course[]>([]);
  isLoading = signal(false);
  message = signal<{ type: "success" | "error"; text: string } | null>(null);

  constructor(private courseService: CourseService) {
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

  async updateCourse(courseId: number, updatedData: Partial<Course>) {
    this.isLoading.set(true);
    this.message.set(null);

    try {
      const updatedCourse = await this.courseService.updateCourse(
        courseId,
        updatedData
      );

      // UI上のデータを更新
      this.courses.update((courses) =>
        courses.map((course) =>
          course.id === courseId ? updatedCourse : course
        )
      );

      // 成功メッセージを表示
      this.message.set({
        type: "success",
        text: `Course "${updatedCourse.title}" has been updated successfully`,
      });

      // 数秒後にメッセージを消す
      setTimeout(() => this.message.set(null), 3000);
    } catch (error) {
      console.error("Failed to update course", error);
      this.message.set({
        type: "error",
        text: "Failed to update course. Please try again.",
      });
    } finally {
      this.isLoading.set(false);
    }
  }
}
```

## UI 更新のための HTML テンプレート

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
        <button (click)="editCourse(course)">Edit</button>
        <button (click)="deleteCourse(course.id)">Delete</button>
      </div>
    </div>
    }
  </div>
</div>
```

## 高度な UI 更新パターン

### 1. オプティミスティック UI 更新

ユーザー体験向上のため、API 応答を待たずに先に UI を更新し、エラー時に元に戻す方法。

```typescript
async updateCourseOptimistic(courseId: number, updatedData: Partial<Course>) {
  // 更新前の状態を保存
  const originalCourses = this.courses();

  // オプティミスティックに先にUIを更新
  this.courses.update(courses =>
    courses.map(course =>
      course.id === courseId ? {...course, ...updatedData} : course
    )
  );

  try {
    // バックグラウンドでAPI呼び出し
    const updatedCourse = await this.courseService.updateCourse(courseId, updatedData);

    // 成功メッセージを表示
    this.message.set({
      type: 'success',
      text: `Course "${updatedCourse.title}" has been updated`
    });

  } catch (error) {
    // エラー時は元の状態に戻す
    this.courses.set(originalCourses);
    this.message.set({
      type: 'error',
      text: 'Failed to update course. Please try again.'
    });
  }
}
```

### 2. 差分更新とアニメーション

変更された要素だけをハイライト表示するアプローチ。

```typescript
async updateCourseWithHighlight(courseId: number, updatedData: Partial<Course>) {
  try {
    const updatedCourse = await this.courseService.updateCourse(courseId, updatedData);

    // 変更されたコースを特定する
    const updatedIndex = this.courses().findIndex(c => c.id === courseId);

    if (updatedIndex !== -1) {
      // UIを更新
      this.courses.update(courses => {
        const newCourses = [...courses];
        newCourses[updatedIndex] = updatedCourse;
        return newCourses;
      });

      // 変更された要素にマーカーを追加（CSSアニメーション用）
      this.highlightedCourseId.set(courseId);

      // 数秒後にハイライトを解除
      setTimeout(() => this.highlightedCourseId.set(null), 2000);
    }

  } catch (error) {
    console.error('Failed to update course', error);
    this.message.set({
      type: 'error',
      text: 'Failed to update course. Please try again.'
    });
  }
}
```

対応する CSS:

```css
.course-card {
  transition: background-color 0.3s ease;
}

.course-card.highlighted {
  animation: highlight-animation 2s ease;
}

@keyframes highlight-animation {
  0% {
    background-color: transparent;
  }
  30% {
    background-color: #fffacd;
  }
  100% {
    background-color: transparent;
  }
}
```

### 3. 編集モードの切り替え

インライン編集のための UI 更新パターン。

```typescript
editingCourseId = signal<number | null>(null);

startEditing(courseId: number) {
  this.editingCourseId.set(courseId);
}

cancelEditing() {
  this.editingCourseId.set(null);
}

async saveInlineEdit(courseId: number, updatedData: Partial<Course>) {
  try {
    const updatedCourse = await this.courseService.updateCourse(courseId, updatedData);

    // UI更新
    this.courses.update(courses =>
      courses.map(course =>
        course.id === courseId ? updatedCourse : course
      )
    );

    // 編集モードを終了
    this.editingCourseId.set(null);

  } catch (error) {
    console.error('Failed to update course', error);
    this.message.set({
      type: 'error',
      text: 'Failed to save changes. Please try again.'
    });
  }
}
```

## 実装のポイント

1. **適切なフィードバック**

   - 処理状態の明示（ローディング、成功、エラー）
   - 一時的なメッセージ表示と自動消去

2. **UI/UX の考慮**

   - オプティミスティック UI 更新による即時反応
   - 視覚的フィードバック（色の変化、アニメーション）

3. **エラーハンドリング**

   - 失敗時の適切なリカバリー
   - ユーザーにわかりやすいエラーメッセージ

4. **パフォーマンス**
   - 必要な部分のみの更新
   - 不要な DOM 操作の最小化

適切な UI 更新戦略を実装することで、データの一貫性を保ちながら、ユーザーに対して即時のフィードバックを提供し、より良いユーザー体験を実現できます。
