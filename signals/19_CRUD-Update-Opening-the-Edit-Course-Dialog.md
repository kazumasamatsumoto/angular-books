# CRUD Update - Opening the Edit Course Dialog

CRUD (Create, Read, Update, Delete) 操作のうち「Update」（更新）処理における編集ダイアログを開く部分についてまとめます。

## 主な実装ステップ

### 1. 編集ダイアログコンポーネントの作成

```typescript
import { Component, inject } from "@angular/core";
import { FormBuilder, FormGroup, Validators } from "@angular/forms";
import { DialogRef } from "@angular/cdk/dialog";
import { Course } from "../models/course.model";

@Component({
  selector: "app-edit-course-dialog",
  templateUrl: "./edit-course-dialog.component.html",
})
export class EditCourseDialogComponent {
  private dialogRef = inject(DialogRef);
  private fb = inject(FormBuilder);

  courseForm: FormGroup;
  course: Course;

  constructor() {
    // ダイアログが開かれるときにデータを受け取る
    this.course = this.dialogRef.data;

    // フォームの初期化
    this.courseForm = this.fb.group({
      title: [this.course.title, [Validators.required]],
      description: [this.course.description, [Validators.required]],
      category: [this.course.category, [Validators.required]],
      level: [this.course.level, [Validators.required]],
      // その他の必要なフォームコントロール
    });
  }

  // 保存ボタンのハンドラー
  save() {
    if (this.courseForm.valid) {
      // フォームの値とコースIDを組み合わせて更新データを作成
      const updatedCourse = {
        ...this.courseForm.value,
        id: this.course.id,
      };

      // ダイアログを閉じて更新データを返す
      this.dialogRef.close(updatedCourse);
    }
  }

  // キャンセルボタンのハンドラー
  cancel() {
    this.dialogRef.close();
  }
}
```

### 2. ダイアログのテンプレート作成

```html
<div class="dialog-container">
  <h2>Edit Course</h2>

  <form [formGroup]="courseForm">
    <div class="form-field">
      <label for="title">Title</label>
      <input id="title" formControlName="title" type="text" />
      <div
        class="error"
        *ngIf="courseForm.get('title')?.invalid && courseForm.get('title')?.touched"
      >
        Title is required
      </div>
    </div>

    <div class="form-field">
      <label for="description">Description</label>
      <textarea
        id="description"
        formControlName="description"
        rows="4"
      ></textarea>
      <div
        class="error"
        *ngIf="courseForm.get('description')?.invalid && courseForm.get('description')?.touched"
      >
        Description is required
      </div>
    </div>

    <div class="form-field">
      <label for="category">Category</label>
      <select id="category" formControlName="category">
        <option value="development">Development</option>
        <option value="design">Design</option>
        <option value="business">Business</option>
      </select>
    </div>

    <div class="form-field">
      <label for="level">Level</label>
      <select id="level" formControlName="level">
        <option value="beginner">Beginner</option>
        <option value="intermediate">Intermediate</option>
        <option value="advanced">Advanced</option>
      </select>
    </div>

    <div class="actions">
      <button type="button" (click)="cancel()">Cancel</button>
      <button type="button" (click)="save()" [disabled]="courseForm.invalid">
        Save
      </button>
    </div>
  </form>
</div>
```

### 3. コース一覧コンポーネントからダイアログを開く

```typescript
import { Component, inject } from "@angular/core";
import { Dialog } from "@angular/cdk/dialog";
import { CourseService } from "../services/course.service";
import { Course } from "../models/course.model";
import { EditCourseDialogComponent } from "./edit-course-dialog.component";

@Component({
  selector: "app-course-list",
  templateUrl: "./course-list.component.html",
})
export class CourseListComponent {
  private dialog = inject(Dialog);
  private courseService = inject(CourseService);

  courses = signal<Course[]>([]);

  ngOnInit() {
    this.loadCourses();
  }

  // コース一覧を読み込む
  async loadCourses() {
    const courses = await this.courseService.getCourses();
    this.courses.set(courses);
  }

  // 編集ダイアログを開く
  openEditDialog(course: Course) {
    const dialogRef = this.dialog.open(EditCourseDialogComponent, {
      width: "500px",
      data: course, // ダイアログにコースデータを渡す
    });

    // ダイアログが閉じられたときの処理
    dialogRef.closed.subscribe((result: Course | undefined) => {
      if (result) {
        // 更新されたコースデータを受け取った場合
        this.updateCourse(result);
      }
    });
  }

  // コースを更新する
  async updateCourse(updatedCourse: Course) {
    try {
      // APIを呼び出してコースを更新
      const result = await this.courseService.updateCourse(
        updatedCourse.id,
        updatedCourse
      );

      if (result) {
        // 成功したら一覧のデータも更新
        this.courses.update((courses) =>
          courses.map((c) => (c.id === updatedCourse.id ? updatedCourse : c))
        );
      }
    } catch (error) {
      console.error("Failed to update course:", error);
      // エラー処理
    }
  }
}
```

### 4. コース一覧のテンプレートに編集ボタンを追加

```html
<div class="course-list">
  @for (course of courses(); track course.id) {
  <div class="course-item">
    <h3>{{ course.title }}</h3>
    <p>{{ course.description }}</p>
    <div class="course-details">
      <span class="category">{{ course.category }}</span>
      <span class="level">{{ course.level }}</span>
    </div>
    <div class="actions">
      <button (click)="openEditDialog(course)">Edit</button>
      <button (click)="deleteCourse(course.id)">Delete</button>
    </div>
  </div>
  }
</div>
```

## 実装のポイント

1. **依存性の注入**

   - Angular 16+ の `inject` 関数を使用し、より簡潔なコード記述が可能

2. **リアクティブフォーム**

   - データバインディングとバリデーションを備えたフォームの実装
   - 既存のコースデータをフォームに初期値として設定

3. **ダイアログデータの受け渡し**

   - ダイアログを開くときにデータを渡し、閉じるときに更新データを受け取る

4. **エラー処理とバリデーション**

   - フォームのバリデーションによるデータ整合性の確保
   - API 呼び出し時のエラーハンドリング

5. **シグナルの活用**
   - リスト状態を `signal()` で管理し、更新時に効率的な UI の再描画を実現

この実装により、ユーザーはコース一覧から編集したいコースの「Edit」ボタンをクリックし、ダイアログで詳細を変更して保存することができます。変更は API を通じてバックエンドに送信され、成功したら即座に一覧表示も更新されます。
