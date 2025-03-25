# CRUD Update - Edit Course Form

CRUD 操作の「Update」部分における編集フォームの実装についてまとめます。このパートではフォームコントロールの設定、バリデーション、データバインディングなどが重要となります。

## 主要コンポーネント

### 1. コースモデルの定義

```typescript
export interface Course {
  id: number;
  title: string;
  description: string;
  category: string;
  level: string;
  duration: number;
  price: number;
  isPublished: boolean;
}
```

### 2. 編集フォームコンポーネント

```typescript
import { Component, OnInit, inject } from "@angular/core";
import { FormBuilder, FormGroup, Validators } from "@angular/forms";
import { ActivatedRoute, Router } from "@angular/router";
import { CourseService } from "../services/course.service";
import { Course } from "../models/course.model";

@Component({
  selector: "app-edit-course",
  templateUrl: "./edit-course.component.html",
  styleUrls: ["./edit-course.component.css"],
})
export class EditCourseComponent implements OnInit {
  private fb = inject(FormBuilder);
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private courseService = inject(CourseService);

  courseId!: number;
  courseForm!: FormGroup;
  isLoading = false;
  errorMessage = "";

  // カテゴリと難易度のオプション
  categories = ["Development", "Design", "Business", "Marketing"];
  levels = ["Beginner", "Intermediate", "Advanced"];

  ngOnInit(): void {
    // URLからコースIDを取得
    this.courseId = +this.route.snapshot.paramMap.get("id")!;

    // フォームの初期化
    this.initForm();

    // コースデータの取得
    this.loadCourse();
  }

  // フォームの初期化
  initForm(): void {
    this.courseForm = this.fb.group({
      title: ["", [Validators.required, Validators.minLength(5)]],
      description: ["", [Validators.required, Validators.minLength(20)]],
      category: ["", Validators.required],
      level: ["", Validators.required],
      duration: [0, [Validators.required, Validators.min(1)]],
      price: [0, [Validators.required, Validators.min(0)]],
      isPublished: [false],
    });
  }

  // コースデータの読み込み
  async loadCourse(): Promise<void> {
    this.isLoading = true;
    try {
      const course = await this.courseService.getCourseById(this.courseId);
      if (course) {
        // フォームに値をセット
        this.courseForm.patchValue(course);
      } else {
        this.errorMessage = "Course not found";
      }
    } catch (error) {
      this.errorMessage = "Failed to load course";
      console.error(error);
    } finally {
      this.isLoading = false;
    }
  }

  // フォーム送信処理
  async onSubmit(): Promise<void> {
    if (this.courseForm.invalid) {
      // フォームが無効な場合はすべてのコントロールにタッチしてエラーを表示
      Object.keys(this.courseForm.controls).forEach((key) => {
        const control = this.courseForm.get(key);
        control?.markAsTouched();
      });
      return;
    }

    this.isLoading = true;
    try {
      const updatedCourse = {
        ...this.courseForm.value,
        id: this.courseId,
      };

      await this.courseService.updateCourse(this.courseId, updatedCourse);
      // 成功したらコース一覧に戻る
      this.router.navigate(["/courses"]);
    } catch (error) {
      this.errorMessage = "Failed to update course";
      console.error(error);
    } finally {
      this.isLoading = false;
    }
  }

  // キャンセルして一覧に戻る
  onCancel(): void {
    this.router.navigate(["/courses"]);
  }

  // 各フォームコントロールにアクセスするためのゲッター
  get title() {
    return this.courseForm.get("title");
  }
  get description() {
    return this.courseForm.get("description");
  }
  get category() {
    return this.courseForm.get("category");
  }
  get level() {
    return this.courseForm.get("level");
  }
  get duration() {
    return this.courseForm.get("duration");
  }
  get price() {
    return this.courseForm.get("price");
  }
}
```

### 3. 編集フォームのテンプレート

```html
<div class="edit-course-container">
  <h2>Edit Course</h2>

  <!-- ローディング表示 -->
  <div class="loading" *ngIf="isLoading">Loading...</div>

  <!-- エラーメッセージ -->
  <div class="error-message" *ngIf="errorMessage">{{ errorMessage }}</div>

  <!-- 編集フォーム -->
  <form
    [formGroup]="courseForm"
    (ngSubmit)="onSubmit()"
    *ngIf="!isLoading && !errorMessage"
  >
    <!-- タイトル -->
    <div class="form-group">
      <label for="title">Title</label>
      <input type="text" id="title" formControlName="title" />
      <div class="validation-error" *ngIf="title?.invalid && title?.touched">
        <span *ngIf="title?.errors?.['required']">Title is required</span>
        <span *ngIf="title?.errors?.['minlength']"
          >Title must be at least 5 characters</span
        >
      </div>
    </div>

    <!-- 説明 -->
    <div class="form-group">
      <label for="description">Description</label>
      <textarea
        id="description"
        formControlName="description"
        rows="4"
      ></textarea>
      <div
        class="validation-error"
        *ngIf="description?.invalid && description?.touched"
      >
        <span *ngIf="description?.errors?.['required']"
          >Description is required</span
        >
        <span *ngIf="description?.errors?.['minlength']"
          >Description must be at least 20 characters</span
        >
      </div>
    </div>

    <!-- カテゴリ -->
    <div class="form-group">
      <label for="category">Category</label>
      <select id="category" formControlName="category">
        <option value="">Select a category</option>
        <option *ngFor="let cat of categories" [value]="cat">{{ cat }}</option>
      </select>
      <div
        class="validation-error"
        *ngIf="category?.invalid && category?.touched"
      >
        Category is required
      </div>
    </div>

    <!-- 難易度 -->
    <div class="form-group">
      <label for="level">Level</label>
      <select id="level" formControlName="level">
        <option value="">Select a level</option>
        <option *ngFor="let lvl of levels" [value]="lvl">{{ lvl }}</option>
      </select>
      <div class="validation-error" *ngIf="level?.invalid && level?.touched">
        Level is required
      </div>
    </div>

    <!-- 期間 -->
    <div class="form-group">
      <label for="duration">Duration (hours)</label>
      <input type="number" id="duration" formControlName="duration" min="1" />
      <div
        class="validation-error"
        *ngIf="duration?.invalid && duration?.touched"
      >
        <span *ngIf="duration?.errors?.['required']">Duration is required</span>
        <span *ngIf="duration?.errors?.['min']"
          >Duration must be at least 1 hour</span
        >
      </div>
    </div>

    <!-- 価格 -->
    <div class="form-group">
      <label for="price">Price</label>
      <input type="number" id="price" formControlName="price" min="0" />
      <div class="validation-error" *ngIf="price?.invalid && price?.touched">
        <span *ngIf="price?.errors?.['required']">Price is required</span>
        <span *ngIf="price?.errors?.['min']">Price cannot be negative</span>
      </div>
    </div>

    <!-- 公開状態 -->
    <div class="form-group checkbox">
      <label>
        <input type="checkbox" formControlName="isPublished" />
        Published
      </label>
    </div>

    <!-- アクションボタン -->
    <div class="form-actions">
      <button type="button" (click)="onCancel()">Cancel</button>
      <button type="submit" [disabled]="courseForm.invalid">
        Save Changes
      </button>
    </div>
  </form>
</div>
```

## 実装のポイント

1. **リアクティブフォーム**

   - FormBuilder を使用した構造化されたフォーム定義
   - コントロールごとの適切なバリデーション

2. **データの取得と設定**

   - ルートパラメータからコース ID 取得
   - 既存のコースデータをフォームに設定（patchValue）

3. **バリデーション処理**

   - 必須入力、最小文字数、最小値など適切なバリデーションの設定
   - エラーメッセージの表示制御

4. **UX 対応**

   - ローディング状態の表示
   - エラーメッセージの表示
   - 無効なフォームの送信防止

5. **送信処理**

   - フォームデータの収集と更新 API の呼び出し
   - 成功時のリダイレクト

6. **使いやすい UI**
   - ドロップダウンリストの活用
   - チェックボックスのシンプルな表現
   - キャンセルボタンの提供

このような編集フォームの実装により、ユーザーは既存のコースデータを安全に更新することができます。バリデーション、ユーザーフィードバック、エラー処理など重要な要素を組み込むことで、使いやすく堅牢なフォームが実現できます。
