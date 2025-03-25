# CRUD Service Layer with async/await and the Angular HTTP Client

Angular の HTTP クライアントを `async/await` 構文と組み合わせた CRUD サービスレイヤーの実装についてまとめます。

## 基本的なサービスの構造

```typescript
import { Injectable } from "@angular/core";
import { HttpClient, HttpErrorResponse } from "@angular/common/http";
import { firstValueFrom } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class CourseService {
  private apiUrl = "api/courses";

  constructor(private http: HttpClient) {}

  // データ取得メソッド (Read)
  async getCourses(): Promise<Course[]> {
    try {
      return await firstValueFrom(this.http.get<Course[]>(this.apiUrl));
    } catch (error) {
      this.handleError(error as HttpErrorResponse);
      return [];
    }
  }

  // 単一データ取得 (Read)
  async getCourseById(id: number): Promise<Course | null> {
    try {
      return await firstValueFrom(
        this.http.get<Course>(`${this.apiUrl}/${id}`)
      );
    } catch (error) {
      this.handleError(error as HttpErrorResponse);
      return null;
    }
  }

  // データ作成 (Create)
  async createCourse(course: Omit<Course, "id">): Promise<Course | null> {
    try {
      return await firstValueFrom(this.http.post<Course>(this.apiUrl, course));
    } catch (error) {
      this.handleError(error as HttpErrorResponse);
      return null;
    }
  }

  // データ更新 (Update)
  async updateCourse(
    id: number,
    course: Partial<Course>
  ): Promise<Course | null> {
    try {
      return await firstValueFrom(
        this.http.put<Course>(`${this.apiUrl}/${id}`, course)
      );
    } catch (error) {
      this.handleError(error as HttpErrorResponse);
      return null;
    }
  }

  // データ削除 (Delete)
  async deleteCourse(id: number): Promise<boolean> {
    try {
      await firstValueFrom(this.http.delete(`${this.apiUrl}/${id}`));
      return true;
    } catch (error) {
      this.handleError(error as HttpErrorResponse);
      return false;
    }
  }

  // エラーハンドリング
  private handleError(error: HttpErrorResponse): void {
    console.error("API Error:", error);
    let errorMessage = "An unknown error occurred";

    if (error.error instanceof ErrorEvent) {
      // クライアントサイドのエラー
      errorMessage = `Error: ${error.error.message}`;
    } else {
      // サーバーサイドのエラー
      errorMessage = `Error Code: ${error.status}, Message: ${error.message}`;
    }

    // エラーをログに記録し、必要に応じてユーザーに通知
    // 実際のアプリケーションでは専用のエラーサービスを使用することが望ましい
    throw new Error(errorMessage);
  }
}
```

## 重要なポイント

### 1. Observable から Promise への変換

Angular の `HttpClient` は Observable を返しますが、`async/await` を使用するためには Promise に変換する必要があります。

```typescript
import { firstValueFrom } from "rxjs";

// Observable から Promise への変換
const courses = await firstValueFrom(this.http.get<Course[]>(this.apiUrl));
```

### 2. エラーハンドリング

非同期処理では try/catch ブロックによるエラー処理が必要です。

```typescript
try {
  return await firstValueFrom(this.http.get<Course[]>(this.apiUrl));
} catch (error) {
  this.handleError(error as HttpErrorResponse);
  return []; // エラー時のフォールバック値
}
```

### 3. 型安全性の確保

TypeScript の型システムを活用して、API レスポンスの型を明示的に指定します。

```typescript
// インターフェースの定義
interface Course {
  id: number;
  title: string;
  description: string;
  // その他のプロパティ
}

// 型指定して HTTP リクエスト
return await firstValueFrom(this.http.get<Course>(`${this.apiUrl}/${id}`));
```

### 4. サービスでの使用例

コンポーネントでこのサービスを使用する場合：

```typescript
@Component({
  selector: "app-course-list",
  template: `...`,
})
export class CourseListComponent {
  courses = signal<Course[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);

  constructor(private courseService: CourseService) {}

  async ngOnInit() {
    this.loading.set(true);
    try {
      const courses = await this.courseService.getCourses();
      this.courses.set(courses);
    } catch (error) {
      this.error.set("Failed to load courses");
    } finally {
      this.loading.set(false);
    }
  }

  async deleteCourse(id: number) {
    const success = await this.courseService.deleteCourse(id);
    if (success) {
      // UI から削除したコースを除外
      this.courses.update((courses) => courses.filter((c) => c.id !== id));
    }
  }
}
```

`async/await` を用いた実装は、コードの可読性を高め、非同期処理のフローをより直感的にします。ただし、RxJS の豊富な機能（リトライ、キャンセル、複数ストリームの結合など）が必要な場合は、従来の Observable ベースのアプローチも検討する価値があります。
