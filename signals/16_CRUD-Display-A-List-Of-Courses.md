# CRUD - Display a List of Courses

以下は Angular で "コース一覧の表示" 機能を実装するためのポイントをまとめたものです。

## 主要コンポーネント

1. **データモデル**

   ```typescript
   export interface Course {
     id: number;
     title: string;
     description: string;
     category: string;
     level: string;
     // その他のプロパティ
   }
   ```

2. **サービス**

   ```typescript
   @Injectable({
     providedIn: "root",
   })
   export class CourseService {
     private apiUrl = "api/courses"; // APIエンドポイント

     constructor(private http: HttpClient) {}

     // コース一覧の取得
     getCourses(): Observable<Course[]> {
       return this.http.get<Course[]>(this.apiUrl);
     }
   }
   ```

3. **コース一覧コンポーネント**

   ```typescript
   @Component({
     selector: "app-course-list",
     templateUrl: "./course-list.component.html",
     styleUrls: ["./course-list.component.css"],
   })
   export class CourseListComponent implements OnInit {
     courses = signal<Course[]>([]);
     loading = signal(false);
     error = signal<string | null>(null);

     constructor(private courseService: CourseService) {}

     ngOnInit(): void {
       this.loadCourses();
     }

     loadCourses(): void {
       this.loading.set(true);
       this.courseService
         .getCourses()
         .pipe(finalize(() => this.loading.set(false)))
         .subscribe({
           next: (data) => this.courses.set(data),
           error: (err) => this.error.set("Failed to load courses"),
         });
     }
   }
   ```

4. **HTML テンプレート**
   ```html
   <div class="container">
     <h2>Courses</h2>

     <!-- ローディング表示 -->
     @if (loading()) {
     <div class="loading">Loading courses...</div>
     }

     <!-- エラー表示 -->
     @if (error()) {
     <div class="error">{{ error() }}</div>
     }

     <!-- コース一覧 -->
     @if (courses().length > 0) {
     <div class="course-grid">
       @for (course of courses(); track course.id) {
       <div class="course-card">
         <h3>{{ course.title }}</h3>
         <p class="category">{{ course.category }}</p>
         <p class="level">Level: {{ course.level }}</p>
         <p class="description">{{ course.description }}</p>
         <div class="actions">
           <button (click)="editCourse(course.id)">Edit</button>
           <button (click)="deleteCourse(course.id)">Delete</button>
         </div>
       </div>
       }
     </div>
     } @else {
     <div class="no-courses">No courses available</div>
     }

     <button class="add-button" (click)="addNewCourse()">Add New Course</button>
   </div>
   ```

## 実装のポイント

1. **シグナルの活用**

   - リストデータを `signal<Course[]>([])` で管理
   - 読み込み状態やエラー状態もシグナルで管理

2. **@if/@for ディレクティブ**

   - 新しい構文を使用して条件付きレンダリングとリスト表示を実装
   - `track` キーワードでパフォーマンスを最適化

3. **エラーハンドリング**

   - ロード失敗時のエラーメッセージ表示
   - `finalize` 演算子でローディング状態の管理

4. **レスポンシブデザイン**

   - CSS Grid または Flexbox を使用して様々な画面サイズに対応

5. **UX の向上**
   - ローディング中の表示
   - データが空の場合の適切なメッセージ
   - 各コースに対する操作ボタン

Angular のシグナル API とコントロールフロー構文の新機能を活用することで、よりクリーンで保守性の高いコース一覧表示機能を実装できます。
