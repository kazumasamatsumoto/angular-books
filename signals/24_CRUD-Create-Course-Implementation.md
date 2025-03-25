# Angular の「Signals」を活用した CRUD 操作 - Create Course 実装について

Angular の新機能である「Signals」を活用したコース作成（Create）機能の実装についてまとめます。

## 概要

Signals は、Angular の最新の状態管理機能で、リアクティブな値の変更を効率的に追跡・伝播します。これを使った Create 操作では、フォーム管理とデータの永続化が効率的に行えます。

## 主要なコンポーネント

1. **Signal 作成とフォーム連携**

   - コース情報を保持する Signal を定義
   - フォーム入力と Signal の双方向バインディング

2. **検証（Validation）**

   - 入力値の検証ロジックを Signal に統合
   - エラー状態も Signal として管理

3. **データ送信と API 連携**

   - フォーム送信時に Signal の値をサービスに渡す
   - 非同期処理の状態も Signal で管理

4. **UI 更新**
   - 送信状態や成功/エラー表示を Signal ベースで制御

## 実装例

```typescript
import { Component, inject, signal } from "@angular/core";
import { CourseService } from "../services/course.service";

@Component({
  selector: "app-course-create",
  template: `
    <form (ngSubmit)="saveCourse()">
      <div>
        <label for="title">タイトル</label>
        <input
          id="title"
          type="text"
          [ngModel]="courseData().title"
          (ngModelChange)="updateTitle($event)"
        />
        <div *ngIf="errors().title">{{ errors().title }}</div>
      </div>

      <div>
        <label for="description">説明</label>
        <textarea
          id="description"
          [ngModel]="courseData().description"
          (ngModelChange)="updateDescription($event)"
        ></textarea>
      </div>

      <button type="submit" [disabled]="isSubmitting() || !isValid()">
        {{ isSubmitting() ? "保存中..." : "コースを作成" }}
      </button>
    </form>
  `,
})
export class CourseCreateComponent {
  private courseService = inject(CourseService);

  // Signals
  courseData = signal({
    title: "",
    description: "",
    category: "development",
  });

  errors = signal<Record<string, string>>({});
  isSubmitting = signal(false);
  isValid = signal(true);

  updateTitle(title: string) {
    this.courseData.update((data) => ({
      ...data,
      title,
    }));
    this.validate();
  }

  updateDescription(description: string) {
    this.courseData.update((data) => ({
      ...data,
      description,
    }));
    this.validate();
  }

  validate() {
    const newErrors: Record<string, string> = {};

    if (!this.courseData().title) {
      newErrors.title = "タイトルは必須です";
    } else if (this.courseData().title.length < 5) {
      newErrors.title = "タイトルは5文字以上必要です";
    }

    this.errors.set(newErrors);
    this.isValid.set(Object.keys(newErrors).length === 0);
  }

  saveCourse() {
    if (!this.isValid()) return;

    this.isSubmitting.set(true);

    this.courseService.createCourse(this.courseData()).subscribe({
      next: () => {
        // 成功処理
        this.isSubmitting.set(false);
        this.courseData.set({
          title: "",
          description: "",
          category: "development",
        });
        // 通知やリダイレクト処理
      },
      error: (err) => {
        // エラー処理
        this.isSubmitting.set(false);
        this.errors.set({
          api: "APIエラー: " + err.message,
        });
      },
    });
  }
}
```

## メリット

1. **リアクティブ性** - データの変更が自動的に UI に反映
2. **効率性** - 必要な部分のみ再描画されるため、パフォーマンスが向上
3. **デバッグのしやすさ** - 状態変化が追跡しやすい
4. **テスト容易性** - Signal は Observable に比べてテストしやすい

## 発展的な使い方

- コンポーネント間での Signal 共有
- computed Signals による派生値の計算
- 複雑なフォームのモジュール化
- アニメーションとの連携

Angular の最新状態管理アプローチである Signals を活用することで、より明示的で効率的な CRUD 操作が実現できます。
