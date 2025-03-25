# Angular の Async Pipe - Observable データをビューに渡すベストプラクティス

Async Pipe (`| async`) は Angular の強力な機能で、Observable や Promise からのデータを簡単にテンプレートにバインドできます。サブスクリプションのライフサイクル管理を自動化し、コードをシンプルにします。

## 基本的な使い方

```typescript
// コンポーネント
import { Component } from "@angular/core";
import { Observable } from "rxjs";
import { DataService } from "./data.service";

@Component({
  selector: "app-user-profile",
  template: `
    <div *ngIf="user$ | async as user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `,
})
export class UserProfileComponent {
  user$: Observable<User>;

  constructor(private dataService: DataService) {
    this.user$ = this.dataService.getUser();
  }
}
```

## Async Pipe の主なメリット

1. **自動サブスクリプション管理**

   - コンポーネントが破棄されると自動的にサブスクリプションを解除
   - `ngOnDestroy` での手動クリーンアップが不要

2. **変更検知の最適化**

   - Observable が新しい値を発行したときだけ変更検知を実行
   - パフォーマンスの向上

3. **コードの簡素化**

   - コンポーネントクラス内のプロパティバインディングコードが減少
   - テンプレート駆動型の宣言的なアプローチ

4. **非同期データの型安全な処理**
   - TypeScript との連携で型安全性が向上

## 応用テクニック

### 1. as 構文を使った一時変数の作成

```html
<div *ngIf="data$ | async as data">
  <!-- data はローカル変数として使用可能 -->
  {{ data.property }}
</div>
```

### 2. ngIf と組み合わせたロード状態の管理

```html
<ng-container *ngIf="users$ | async as users; else loading">
  <user-list [users]="users"></user-list>
</ng-container>

<ng-template #loading>
  <loading-spinner></loading-spinner>
</ng-template>
```

### 3. 複数の Observable の組み合わせ

```typescript
// コンポーネント内
import { combineLatest } from "rxjs";

// プロパティ定義
viewModel$ = combineLatest([
  this.userService.getCurrentUser(),
  this.productService.getProducts(),
]).pipe(map(([user, products]) => ({ user, products })));
```

```html
<!-- テンプレート -->
<div *ngIf="viewModel$ | async as vm">
  <h2>Welcome, {{ vm.user.name }}</h2>
  <product-list [products]="vm.products"></product-list>
</div>
```

### 4. Observable の配列を扱う

```html
<div *ngFor="let item of items$ | async">{{ item.name }}</div>
```

## エラー処理とロード状態

より完全なエラー処理とロード状態の管理:

```typescript
// RxJS オペレータを使用して UI 状態を管理
interface State<T> {
  loading: boolean;
  data?: T;
  error?: string;
}

// コンポーネント内
data$ = this.dataService.getData().pipe(
  map((data) => ({ loading: false, data })),
  catchError((error) => of({ loading: false, error: error.message })),
  startWith({ loading: true })
);
```

```html
<!-- テンプレート -->
<ng-container *ngIf="data$ | async as state">
  <loading-spinner *ngIf="state.loading"></loading-spinner>

  <error-message *ngIf="state.error" [message]="state.error"></error-message>

  <data-display *ngIf="state.data" [data]="state.data"></data-display>
</ng-container>
```

## 最新の Angular での発展的な使い方

Angular の最新バージョンでは、Async Pipe とシグナルを組み合わせた新しいパターンも登場しています:

```typescript
// Angular 16+
import { Component, inject, computed } from "@angular/core";
import { toSignal } from "@angular/core/rxjs-interop";

@Component({
  // ...
})
export class ModernComponent {
  private dataService = inject(DataService);

  // Observable を Signal に変換
  private userData = toSignal(this.dataService.getUser());

  // 派生した Signal
  userGreeting = computed(() => {
    const user = this.userData();
    return user ? `Hello, ${user.name}!` : "Loading...";
  });
}
```

## パフォーマンスのヒント

- 同じ Observable を複数回使用する場合は、ローカル変数として保存する
- 大きなデータセットには `trackBy` を使用する
- 複雑な計算には `pipe()` 内で処理を行い、テンプレートはシンプルに保つ

## よくある間違い

1. **同じ Observable を複数回使用**

   ```html
   <!-- 悪い例: 毎回評価される -->
   <div>{{ users$ | async | json }}</div>
   <div>Count: {{ (users$ | async)?.length }}</div>

   <!-- 良い例 -->
   <div *ngIf="users$ | async as users">
     <div>{{ users | json }}</div>
     <div>Count: {{ users.length }}</div>
   </div>
   ```

2. **サブスクリプションの混在**
   ```typescript
   // 悪い例: 手動サブスクリプションと async pipe の混在
   ngOnInit() {
     this.subscription = this.data$.subscribe(data => this.localData = data);
   }
   ```

Async Pipe は、Angular アプリケーションでの非同期データ処理を大幅に簡素化するベストプラクティスであり、特に RxJS と組み合わせることで強力なデータフローパターンを実現できます。

# Async Pipe の使用シナリオ

Async Pipe は様々なシナリオで役立ちます。以下に代表的なユースケースをご紹介します：

## 1. HTTP リクエストの結果表示

```typescript
@Component({
  selector: "app-users",
  template: `
    <div *ngIf="users$ | async as users; else loading">
      <user-card *ngFor="let user of users" [user]="user"></user-card>
    </div>
    <ng-template #loading>データを読み込み中...</ng-template>
  `,
})
export class UsersComponent {
  users$ = this.http.get<User[]>("/api/users");
  constructor(private http: HttpClient) {}
}
```

## 2. リアルタイムデータの表示（Firebase/WebSocket など）

```typescript
@Component({
  selector: "app-chat",
  template: `
    <div class="messages">
      <div *ngFor="let message of messages$ | async" class="message">
        {{ message.text }}
      </div>
    </div>
  `,
})
export class ChatComponent {
  messages$ = this.firestore.collection("messages").valueChanges();
  constructor(private firestore: AngularFirestore) {}
}
```

## 3. フォーム入力のリアクティブな処理

```typescript
@Component({
  selector: "app-search",
  template: `
    <input [formControl]="searchControl" />
    <div *ngIf="searchResults$ | async as results">
      <div *ngFor="let result of results">{{ result.name }}</div>
    </div>
  `,
})
export class SearchComponent {
  searchControl = new FormControl("");

  searchResults$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    switchMap((term) => this.searchService.search(term))
  );

  constructor(private searchService: SearchService) {}
}
```

## 4. ルートパラメータに基づいたデータ取得

```typescript
@Component({
  selector: 'app-product-detail',
  template: `
    <div *ngIf="product$ | async as product">
      <h1>{{ product.name }}</h1>
      <p>{{ product.description }}</p>
      <span>${{ product.price }}</span>
    </div>
  `
})
export class ProductDetailComponent {
  product$ = this.route.paramMap.pipe(
    map(params => params.get('id')),
    switchMap(id => this.productService.getProduct(id))
  );

  constructor(
    private route: ActivatedRoute,
    private productService: ProductService
  ) {}
}
```

## 5. 状態管理（NgRx/RxJS ストア）

```typescript
@Component({
  selector: "app-counter",
  template: `
    <div>Current Count: {{ count$ | async }}</div>
    <button (click)="increment()">+</button>
    <button (click)="decrement()">-</button>
  `,
})
export class CounterComponent {
  count$ = this.store.select((state) => state.counter.count);

  constructor(private store: Store) {}

  increment() {
    this.store.dispatch(increment());
  }

  decrement() {
    this.store.dispatch(decrement());
  }
}
```

## 6. 複数のデータストリームの組み合わせ

```typescript
@Component({
  selector: "app-dashboard",
  template: `
    <div *ngIf="dashboardData$ | async as data">
      <user-info [user]="data.user"></user-info>
      <recent-orders [orders]="data.orders"></recent-orders>
      <notifications [count]="data.notificationCount"></notifications>
    </div>
  `,
})
export class DashboardComponent {
  user$ = this.userService.getCurrentUser();
  orders$ = this.orderService.getRecentOrders();
  notifications$ = this.notificationService.getUnreadCount();

  dashboardData$ = combineLatest([
    this.user$,
    this.orders$,
    this.notifications$,
  ]).pipe(
    map(([user, orders, notificationCount]) => ({
      user,
      orders,
      notificationCount,
    }))
  );

  constructor(
    private userService: UserService,
    private orderService: OrderService,
    private notificationService: NotificationService
  ) {}
}
```

## 7. 条件付きローディングやアクセス制御

```typescript
@Component({
  selector: "app-admin-panel",
  template: `
    <ng-container *ngIf="userPermissions$ | async as permissions">
      <admin-dashboard
        *ngIf="permissions.isAdmin; else unauthorized"
      ></admin-dashboard>
      <ng-template #unauthorized>
        <div>管理者権限がありません</div>
      </ng-template>
    </ng-container>
  `,
})
export class AdminPanelComponent {
  userPermissions$ = this.authService.getCurrentUserPermissions();
  constructor(private authService: AuthService) {}
}
```

## 8. ポーリングによる定期的なデータ更新

```typescript
@Component({
  selector: "app-status",
  template: ` <div>システム状態: {{ status$ | async }}</div> `,
})
export class StatusComponent implements OnInit {
  status$ = interval(30000).pipe(
    startWith(0),
    switchMap(() => this.statusService.getStatus())
  );

  constructor(private statusService: StatusService) {}
}
```

## 9. 非同期バリデーションの結果表示

```typescript
@Component({
  selector: "app-username-form",
  template: `
    <input [formControl]="usernameControl" />
    <div *ngIf="usernameAvailable$ | async">このユーザー名は利用可能です</div>
  `,
})
export class UsernameFormComponent {
  usernameControl = new FormControl("");

  usernameAvailable$ = this.usernameControl.valueChanges.pipe(
    debounceTime(300),
    switchMap((username) =>
      this.userService.checkUsernameAvailability(username)
    )
  );

  constructor(private userService: UserService) {}
}
```

Async Pipe は非同期データフローを扱う多くのシナリオで役立ち、コードを簡潔かつ安全に保ちます。コンポーネントのライフサイクルに自動的に適応するため、リソースリークを防ぎながら宣言的なプログラミングを実現できます。
