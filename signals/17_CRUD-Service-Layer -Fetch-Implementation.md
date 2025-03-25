# CRUD Service Layer - Fetch Implementation

Angular でのサービスレイヤーにおける Fetch 実装（データ取得）についてまとめます。これは CRUD（Create, Read, Update, Delete）操作の「Read」部分に相当します。

## 基本構造

```typescript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable, catchError, map, throwError } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class CourseService {
  private apiUrl = "api/courses"; // APIエンドポイント

  constructor(private http: HttpClient) {}

  // コース一覧を取得するメソッド
  getCourses(): Observable<Course[]> {
    return this.http
      .get<Course[]>(this.apiUrl)
      .pipe(catchError(this.handleError));
  }

  // 特定のコースを取得するメソッド
  getCourseById(id: number): Observable<Course> {
    const url = `${this.apiUrl}/${id}`;
    return this.http.get<Course>(url).pipe(catchError(this.handleError));
  }

  // エラーハンドリング
  private handleError(error: any) {
    console.error("An error occurred:", error);
    return throwError(
      () => new Error("Something went wrong; please try again later.")
    );
  }
}
```

## 実装のポイント

1. **HttpClient の活用**

   - Angular の HttpClient を使用して HTTP リクエストを送信
   - 型パラメータで戻り値の型を指定 (`get<Course[]>`)

2. **Observable ベースの設計**

   - 非同期データフローを Observable で管理
   - コンポーネントでの購読が容易

3. **エラーハンドリング**

   - `catchError` 演算子でエラーをキャッチ
   - 共通のエラーハンドリング関数の実装

4. **パラメータ付きリクエスト**

   - ID や検索条件などのパラメータを URL に含める
   - より複雑な検索条件は HttpParams を使用

5. **レスポンス処理**
   - 必要に応じて `map` 演算子でレスポンスを変換

## 高度な実装例

### フィルタリングと並べ替え機能

```typescript
getCoursesWithFilters(category?: string, sortBy?: string): Observable<Course[]> {
  let params = new HttpParams();

  if (category) {
    params = params.append('category', category);
  }

  if (sortBy) {
    params = params.append('sortBy', sortBy);
  }

  return this.http.get<Course[]>(this.apiUrl, { params })
    .pipe(
      catchError(this.handleError)
    );
}
```

### ページネーション対応

```typescript
getPagedCourses(page: number, pageSize: number): Observable<PagedResponse<Course>> {
  const params = new HttpParams()
    .append('page', page.toString())
    .append('pageSize', pageSize.toString());

  return this.http.get<PagedResponse<Course>>(this.apiUrl, { params })
    .pipe(
      catchError(this.handleError)
    );
}

// ページネーション用のレスポンス型
interface PagedResponse<T> {
  items: T[];
  totalCount: number;
  pageCount: number;
  currentPage: number;
}
```

### キャッシュの実装

```typescript
private coursesCache = signal<Course[] | null>(null);

getCourses(): Observable<Course[]> {
  // キャッシュがあればそれを返す
  if (this.coursesCache()) {
    return of(this.coursesCache()!);
  }

  // キャッシュがなければAPIから取得
  return this.http.get<Course[]>(this.apiUrl)
    .pipe(
      tap(courses => this.coursesCache.set(courses)),
      catchError(this.handleError)
    );
}

// キャッシュを無効化
invalidateCache(): void {
  this.coursesCache.set(null);
}
```

サービスレイヤーでのデータ取得実装は、アプリケーションのビジネスロジックと HTTP 通信を分離し、コンポーネントを簡潔に保つ重要な役割を果たします。適切なキャッシュ、エラーハンドリング、データ変換を実装することで、効率的なデータフローを実現できます。
