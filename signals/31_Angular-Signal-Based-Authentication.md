# Angular Signal-Based Authentication

## 概要

Signal-Based Authentication（シグナルベース認証）は、Angular の新機能である Signals を使用して認証状態を管理する手法です。従来の RxJS ベースの認証システムと比較して、より直感的で効率的な状態管理を実現します。

## 主要コンポーネント

1. **認証状態のシグナル**

   - ユーザー情報や認証状態を保持するシグナル
   - アプリケーション全体での一貫した認証状態管理

2. **トークン管理**

   - JWT などの認証トークンの管理
   - トークンの保存、更新、削除のロジック

3. **認証ガード機能**
   - 保護されたルートへのアクセス制御
   - シグナルベースの権限チェック

## 実装例

### 認証サービスの作成

```typescript
// auth.service.ts
import { Injectable, signal, computed, effect, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Router } from "@angular/router";
import { catchError, tap } from "rxjs/operators";
import { of } from "rxjs";

export interface User {
  id: string;
  email: string;
  name: string;
  roles: string[];
}

export interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  loading: boolean;
  error: string | null;
}

@Injectable({
  providedIn: "root",
})
export class AuthService {
  private http = inject(HttpClient);
  private router = inject(Router);

  // 認証状態を管理する非公開シグナル
  private authStateSignal = signal<AuthState>({
    user: null,
    token: null,
    isAuthenticated: false,
    loading: false,
    error: null,
  });

  // 公開されたcomputed values
  readonly user = computed(() => this.authStateSignal().user);
  readonly isAuthenticated = computed(
    () => this.authStateSignal().isAuthenticated
  );
  readonly isLoading = computed(() => this.authStateSignal().loading);
  readonly authError = computed(() => this.authStateSignal().error);

  // 特定のロールを持っているかチェックするcomputed
  hasRole(role: string) {
    return computed(() => {
      const user = this.user();
      return user && user.roles.includes(role);
    });
  }

  constructor() {
    // 初期化時にローカルストレージからトークンを復元
    this.restoreAuthState();

    // トークン変更時の処理をeffectで設定
    effect(() => {
      const token = this.authStateSignal().token;
      if (token) {
        localStorage.setItem("auth_token", token);
      } else {
        localStorage.removeItem("auth_token");
      }
    });
  }

  // ログイン処理
  login(email: string, password: string) {
    // ローディング状態を設定
    this.authStateSignal.update((state) => ({
      ...state,
      loading: true,
      error: null,
    }));

    return this.http
      .post<{ token: string; user: User }>("/api/auth/login", {
        email,
        password,
      })
      .pipe(
        tap((response) => {
          // 認証成功時の状態更新
          this.authStateSignal.set({
            user: response.user,
            token: response.token,
            isAuthenticated: true,
            loading: false,
            error: null,
          });
        }),
        catchError((error) => {
          // エラー時の状態更新
          this.authStateSignal.update((state) => ({
            ...state,
            loading: false,
            error: error.error?.message || "ログインに失敗しました",
          }));
          return of(null);
        })
      );
  }

  // ログアウト処理
  logout() {
    // 認証状態をリセット
    this.authStateSignal.set({
      user: null,
      token: null,
      isAuthenticated: false,
      loading: false,
      error: null,
    });

    // ホーム画面などへリダイレクト
    this.router.navigate(["/login"]);
  }

  // トークン検証と更新
  validateToken() {
    const token = this.authStateSignal().token;

    if (!token) {
      return of(false);
    }

    return this.http.get<{ user: User }>("/api/auth/validate-token").pipe(
      tap((response) => {
        // ユーザー情報の更新
        this.authStateSignal.update((state) => ({
          ...state,
          user: response.user,
          isAuthenticated: true,
        }));
      }),
      catchError(() => {
        // トークンが無効な場合、認証状態をリセット
        this.logout();
        return of(false);
      })
    );
  }

  // ローカルストレージからの認証状態復元
  private restoreAuthState() {
    const token = localStorage.getItem("auth_token");

    if (token) {
      // トークンがあれば認証状態を設定
      this.authStateSignal.update((state) => ({
        ...state,
        token,
        isAuthenticated: true,
        loading: true,
      }));

      // バックエンドでトークンを検証
      this.validateToken().subscribe();
    }
  }
}
```

### 認証ガードの実装

```typescript
// auth.guard.ts
import { inject } from "@angular/core";
import { Router, CanActivateFn } from "@angular/router";
import { AuthService } from "./auth.service";

// CanActivate関数型のガード（Angular 14+）
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // 未認証の場合はログインページへリダイレクト
  router.navigate(["/login"]);
  return false;
};

// 特定のロールが必要なルート用のガード
export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  const requiredRole = route.data["requiredRole"];

  if (authService.isAuthenticated() && authService.hasRole(requiredRole)()) {
    return true;
  }

  // 権限不足の場合は403ページなどへリダイレクト
  router.navigate(["/forbidden"]);
  return false;
};
```

### HTTP Interceptor の実装

```typescript
// auth.interceptor.ts
import { Injectable } from "@angular/core";
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
} from "@angular/common/http";
import { Observable } from "rxjs";
import { AuthService } from "./auth.service";

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // 現在の認証トークンを取得
    const token = this.authService["authStateSignal"]().token;

    // トークンがある場合はリクエストヘッダーに追加
    if (token) {
      request = request.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`,
        },
      });
    }

    return next.handle(request);
  }
}
```

### ログインコンポーネント

```typescript
// login.component.ts
import { Component, inject } from "@angular/core";
import { FormBuilder, FormGroup, Validators } from "@angular/forms";
import { Router } from "@angular/router";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-login",
  template: `
    <div class="login-container">
      <h2>ログイン</h2>

      <div *ngIf="authService.authError()" class="error-message">
        {{ authService.authError() }}
      </div>

      <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
        <div class="form-group">
          <label for="email">メールアドレス</label>
          <input id="email" type="email" formControlName="email" />
          <div
            *ngIf="email?.invalid && (email?.dirty || email?.touched)"
            class="error"
          >
            メールアドレスを正しく入力してください
          </div>
        </div>

        <div class="form-group">
          <label for="password">パスワード</label>
          <input id="password" type="password" formControlName="password" />
          <div
            *ngIf="password?.invalid && (password?.dirty || password?.touched)"
            class="error"
          >
            パスワードを入力してください
          </div>
        </div>

        <button
          type="submit"
          [disabled]="loginForm.invalid || authService.isLoading()"
        >
          {{ authService.isLoading() ? "ログイン中..." : "ログイン" }}
        </button>
      </form>
    </div>
  `,
})
export class LoginComponent {
  private fb = inject(FormBuilder);
  private router = inject(Router);
  authService = inject(AuthService);

  loginForm: FormGroup = this.fb.group({
    email: ["", [Validators.required, Validators.email]],
    password: ["", Validators.required],
  });

  get email() {
    return this.loginForm.get("email");
  }
  get password() {
    return this.loginForm.get("password");
  }

  onSubmit() {
    if (this.loginForm.invalid) return;

    const { email, password } = this.loginForm.value;

    this.authService.login(email, password).subscribe((response) => {
      if (response) {
        this.router.navigate(["/dashboard"]);
      }
    });
  }
}
```

### ユーザープロフィールコンポーネント

```typescript
// profile.component.ts
import { Component, inject } from "@angular/core";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-profile",
  template: `
    <div class="profile-container" *ngIf="authService.user() as user">
      <h2>プロフィール</h2>

      <div class="user-info">
        <p><strong>名前:</strong> {{ user.name }}</p>
        <p><strong>メール:</strong> {{ user.email }}</p>
        <p><strong>ロール:</strong> {{ user.roles.join(", ") }}</p>
      </div>

      <button (click)="authService.logout()">ログアウト</button>
    </div>
  `,
})
export class ProfileComponent {
  authService = inject(AuthService);
}
```

### ナビゲーションバーコンポーネント

```typescript
// nav.component.ts
import { Component, inject } from "@angular/core";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-nav",
  template: `
    <nav class="main-nav">
      <div class="nav-brand">
        <a routerLink="/">Angular Auth App</a>
      </div>

      <div class="nav-links">
        <a routerLink="/home">ホーム</a>

        <!-- 認証状態によって表示を切り替え -->
        @if (authService.isAuthenticated()) {
        <a routerLink="/dashboard">ダッシュボード</a>
        <a routerLink="/profile">プロフィール</a>

        <!-- 管理者のみ表示 -->
        @if (authService.hasRole('admin')()) {
        <a routerLink="/admin">管理画面</a>
        }

        <button (click)="authService.logout()">ログアウト</button>
        } @else {
        <a routerLink="/login">ログイン</a>
        <a routerLink="/register">新規登録</a>
        }
      </div>
    </nav>
  `,
})
export class NavComponent {
  authService = inject(AuthService);
}
```

### ルーティング設定

```typescript
// app-routing.module.ts
import { NgModule } from "@angular/core";
import { RouterModule, Routes } from "@angular/router";
import { HomeComponent } from "./components/home/home.component";
import { LoginComponent } from "./components/login/login.component";
import { DashboardComponent } from "./components/dashboard/dashboard.component";
import { ProfileComponent } from "./components/profile/profile.component";
import { AdminComponent } from "./components/admin/admin.component";
import { authGuard, roleGuard } from "./guards/auth.guard";

const routes: Routes = [
  { path: "", component: HomeComponent },
  { path: "login", component: LoginComponent },
  {
    path: "dashboard",
    component: DashboardComponent,
    canActivate: [authGuard],
  },
  {
    path: "profile",
    component: ProfileComponent,
    canActivate: [authGuard],
  },
  {
    path: "admin",
    component: AdminComponent,
    canActivate: [roleGuard],
    data: { requiredRole: "admin" },
  },
  { path: "**", redirectTo: "" },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

## 高度な機能

### トークン更新機能

```typescript
// auth.service.ts に追加
// リフレッシュトークンを使用した自動更新
refreshToken() {
  const token = this.authStateSignal().token;

  if (!token) {
    return of(false);
  }

  return this.http.post<{token: string}>('/api/auth/refresh-token', {})
    .pipe(
      tap(response => {
        // 新しいトークンで認証状態を更新
        this.authStateSignal.update(state => ({
          ...state,
          token: response.token
        }));
      }),
      catchError(() => {
        // リフレッシュに失敗した場合はログアウト
        this.logout();
        return of(false);
      })
    );
}
```

### セッションタイムアウト管理

```typescript
// auth.service.ts に追加
private sessionTimeoutId: any;

// トークンからタイムアウト時間を計算
private setupSessionTimeout() {
  const token = this.authStateSignal().token;

  if (!token) return;

  // トークンからexpiration timeを取得（JWT形式を想定）
  const tokenParts = token.split('.');
  if (tokenParts.length !== 3) return;

  try {
    const tokenPayload = JSON.parse(atob(tokenParts[1]));
    const expiresAt = tokenPayload.exp * 1000; // UNIXタイムスタンプをミリ秒に変換
    const timeoutDuration = expiresAt - Date.now() - 60000; // 1分前に更新

    // 既存のタイムアウトをクリア
    if (this.sessionTimeoutId) {
      clearTimeout(this.sessionTimeoutId);
    }

    // 新しいタイムアウトを設定
    if (timeoutDuration > 0) {
      this.sessionTimeoutId = setTimeout(() => {
        this.refreshToken().subscribe();
      }, timeoutDuration);
    } else {
      // 既に期限切れの場合は即座に更新
      this.refreshToken().subscribe();
    }
  } catch (e) {
    console.error('トークンの解析に失敗しました', e);
  }
}
```

### ソーシャルログイン統合

```typescript
// auth.service.ts に追加
// ソーシャルログイン（Google）
loginWithGoogle() {
  // ローディング状態を設定
  this.authStateSignal.update(state => ({
    ...state,
    loading: true,
    error: null
  }));

  // 新しいウィンドウでGoogleログインページを開く
  const googleAuthUrl = '/api/auth/google';
  const width = 500;
  const height = 600;
  const left = window.screenX + (window.outerWidth - width) / 2;
  const top = window.screenY + (window.outerHeight - height) / 2;

  const popup = window.open(
    googleAuthUrl,
    'googleLogin',
    `width=${width},height=${height},left=${left},top=${top}`
  );

  // ウィンドウのメッセージイベントをリッスン
  window.addEventListener('message', (event) => {
    if (event.origin !== window.location.origin) return;

    if (event.data.type === 'social_auth_success') {
      const { token, user } = event.data;

      this.authStateSignal.set({
        user,
        token,
        isAuthenticated: true,
        loading: false,
        error: null
      });

      popup?.close();
      this.router.navigate(['/dashboard']);
    } else if (event.data.type === 'social_auth_error') {
      this.authStateSignal.update(state => ({
        ...state,
        loading: false,
        error: 'ソーシャルログインに失敗しました'
      }));

      popup?.close();
    }
  });
}
```

## メリット

1. **リアクティブな状態管理** - Signals による効率的な UI 更新
2. **コード簡略化** - RxJS の Observable チェーンに比べて直感的
3. **デバッグのしやすさ** - 状態変化の追跡が容易
4. **パフォーマンス向上** - 必要な部分のみ更新される細粒度リアクティビティ
5. **TypeSafe な実装** - タイプチェックによる安全性

## ベストプラクティス

1. **状態の集中管理** - 認証関連の状態を単一のサービスで管理する
2. **セキュリティ考慮** - トークンの安全な保存と更新の自動化
3. **ユーザーエクスペリエンス向上** - セッションタイムアウトの適切な処理
4. **エラー処理の強化** - 認証エラーの明確な提示とハンドリング
5. **効率的なルート保護** - 細かな権限制御によるセキュリティ向上

Angular Signals を活用した認証システムは、より明確でメンテナンスしやすいコードベースを提供し、ユーザー認証の管理を効率化します。特に大規模なアプリケーションでは、状態管理の簡素化とパフォーマンスの向上が顕著になります。
