# CRUD Update - Triggering the HTTP PUT Call

CRUD 操作の「Update」部分における、HTTP リクエスト（PUT）の実行方法についてまとめます。

## HTTP PUT リクエストの基本

```typescript
import { Injectable } from "@angular/core";
import { HttpClient, HttpErrorResponse } from "@angular/common/http";
import { Observable, catchError, throwError } from "rxjs";
import { Course } from "../models/course.model";

@Injectable({
  providedIn: "root",
})
export class CourseService {
  private apiUrl = "api/courses"; // APIのエンドポイント

  constructor(private http: HttpClient) {}

  // Observable を返すバージョン
  updateCourse(id: number, courseData: Partial<Course>): Observable<Course> {
    const url = `${this.apiUrl}/${id}`;
    return this.http
      .put<Course>(url, courseData)
      .pipe(catchError(this.handleError));
  }

  // async/await で使用するバージョン
  async updateCourseAsync(
    id: number,
    courseData: Partial<Course>
  ): Promise<Course> {
    const url = `${this.apiUrl}/${id}`;
    try {
      return await firstValueFrom(this.http.put<Course>(url, courseData));
    } catch (error) {
      throw this.handleError(error as HttpErrorResponse);
    }
  }

  private handleError(error: HttpErrorResponse) {
    // エラー処理のロジック
    return throwError(
      () => new Error("Something went wrong; please try again later.")
    );
  }
}
```

## コンポーネントから PUT リクエストを実行する方法

### 1. Observable アプローチ

```typescript
import { Component } from "@angular/core";
import { CourseService } from "../services/course.service";

@Component({
  selector: "app-edit-course",
  templateUrl: "./edit-course.component.html",
})
export class EditCourseComponent {
  courseId = 123; // 実際には動的に取得
  isSubmitting = false;
  errorMessage = "";

  // コース更新データ
  updateData = {
    title: "Updated Angular Course",
    description: "Learn Angular with the latest features",
    // その他の更新フィールド
  };

  constructor(private courseService: CourseService) {}

  onSaveChanges() {
    this.isSubmitting = true;
    this.errorMessage = "";

    this.courseService.updateCourse(this.courseId, this.updateData).subscribe({
      next: (updatedCourse) => {
        console.log("Course updated successfully", updatedCourse);
        // 成功時の処理（リダイレクトなど）
        this.isSubmitting = false;
      },
      error: (error) => {
        console.error("Error updating course", error);
        this.errorMessage = "Failed to update course";
        this.isSubmitting = false;
      },
    });
  }
}
```

### 2. async/await アプローチ

```typescript
import { Component } from "@angular/core";
import { CourseService } from "../services/course.service";

@Component({
  selector: "app-edit-course",
  templateUrl: "./edit-course.component.html",
})
export class EditCourseComponent {
  courseId = 123; // 実際には動的に取得
  isSubmitting = false;
  errorMessage = "";

  // コース更新データ
  updateData = {
    title: "Updated Angular Course",
    description: "Learn Angular with the latest features",
    // その他の更新フィールド
  };

  constructor(private courseService: CourseService) {}

  async onSaveChanges() {
    this.isSubmitting = true;
    this.errorMessage = "";

    try {
      const updatedCourse = await this.courseService.updateCourseAsync(
        this.courseId,
        this.updateData
      );

      console.log("Course updated successfully", updatedCourse);
      // 成功時の処理（リダイレクトなど）
    } catch (error) {
      console.error("Error updating course", error);
      this.errorMessage = "Failed to update course";
    } finally {
      this.isSubmitting = false;
    }
  }
}
```

### 3. シグナル API を活用したアプローチ

```typescript
import { Component, signal } from "@angular/core";
import { CourseService } from "../services/course.service";
import { Course } from "../models/course.model";

@Component({
  selector: "app-edit-course",
  templateUrl: "./edit-course.component.html",
})
export class EditCourseComponent {
  courseId = 123; // 実際には動的に取得

  // 状態管理にシグナルを使用
  isSubmitting = signal(false);
  errorMessage = signal<string | null>(null);
  courseData = signal<Partial<Course>>({
    title: "Updated Angular Course",
    description: "Learn Angular with the latest features",
    // その他の更新フィールド
  });

  constructor(private courseService: CourseService) {}

  async onSaveChanges() {
    this.isSubmitting.set(true);
    this.errorMessage.set(null);

    try {
      const updatedCourse = await this.courseService.updateCourseAsync(
        this.courseId,
        this.courseData()
      );

      console.log("Course updated successfully", updatedCourse);
      // 成功時の処理（リダイレクトなど）
    } catch (error) {
      console.error("Error updating course", error);
      this.errorMessage.set("Failed to update course");
    } finally {
      this.isSubmitting.set(false);
    }
  }
}
```

## 実装のポイント

1. **リクエストヘッダーの設定**

```typescript
updateCourse(id: number, courseData: Partial<Course>): Observable<Course> {
  const url = `${this.apiUrl}/${id}`;
  const headers = { 'Content-Type': 'application/json' };

  return this.http.put<Course>(url, courseData, { headers })
    .pipe(
      catchError(this.handleError)
    );
}
```

2. **進捗状況の追跡**

```typescript
updateCourse(id: number, courseData: Partial<Course>): Observable<Course> {
  const url = `${this.apiUrl}/${id}`;

  return this.http.put<Course>(url, courseData, {
    reportProgress: true,
    observe: 'events'
  }).pipe(
    map(event => this.getEventMessage(event, courseData)),
    catchError(this.handleError)
  );
}

private getEventMessage(event: HttpEvent<any>, courseData: any) {
  switch (event.type) {
    case HttpEventType.UploadProgress:
      const percentDone = event.total ? Math.round(100 * event.loaded / event.total) : 0;
      return { status: 'progress', message: `${percentDone}%` };

    case HttpEventType.Response:
      return { status: 'success', data: event.body };

    default:
      return { status: 'pending' };
  }
}
```

3. **リトライロジックの追加**

```typescript
import { retry, catchError } from 'rxjs/operators';

updateCourse(id: number, courseData: Partial<Course>): Observable<Course> {
  const url = `${this.apiUrl}/${id}`;

  return this.http.put<Course>(url, courseData)
    .pipe(
      retry(3), // 失敗時に最大3回まで再試行
      catchError(this.handleError)
    );
}
```

4. **異なるレスポンス形式への対応**

```typescript
updateCourse(id: number, courseData: Partial<Course>): Observable<Course> {
  const url = `${this.apiUrl}/${id}`;

  return this.http.put<any>(url, courseData)
    .pipe(
      map(response => {
        // APIのレスポンス形式に応じてマッピング
        if (response.data) {
          return response.data as Course;
        }
        return response as Course;
      }),
      catchError(this.handleError)
    );
}
```

HTTP PUT リクエストを適切に実装することで、バックエンドサーバーとの効率的なデータ同期が可能になり、ユーザー体験の向上につながります。エラー処理やローディング状態の表示も含めた包括的な実装が重要です。
