# Angular の認証 Signal-Based サービス

Angular の最新バージョンでは、Signal API を活用した認証サービスを実装できるようになりました。これにより、リアクティブでパフォーマンスに優れた認証システムを構築できます。

## Signal-Based 認証サービスの主な特徴

1. **リアクティブな状態管理**

   - ユーザーの認証状態をシグナルとして管理
   - 認証状態の変化を自動的に検知して関連コンポーネントを更新

2. **最適化されたレンダリング**

   - 必要なコンポーネントのみを再レンダリング
   - パフォーマンスの向上とメモリ使用量の最適化

3. **型安全性**
   - TypeScript との完全な統合
   - コンパイル時のエラーチェック

## 基本的な実装例

以下に、Signal-Based 認証サービスの基本的な実装例を示します：

```typescript
import { Injectable, signal, computed } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Router } from "@angular/router";
import { tap, catchError } from "rxjs/operators";
import { of } from "rxjs";

export interface User {
  id: string;
  email: string;
  name: string;
}

export interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
}

@Injectable({
  providedIn: "root",
})
export class AuthService {
  // 認証状態をシグナルとして定義
  private authState = signal<AuthState>({
    user: null,
    token: null,
    isAuthenticated: false,
  });

  // 派生シグナル
  readonly user = computed(() => this.authState().user);
  readonly token = computed(() => this.authState().token);
  readonly isAuthenticated = computed(() => this.authState().isAuthenticated);

  constructor(private http: HttpClient, private router: Router) {
    // ローカルストレージから認証状態を復元
    this.restoreAuthState();
  }

  login(email: string, password: string) {
    return this.http
      .post<{ user: User; token: string }>("/api/login", { email, password })
      .pipe(
        tap((response) => {
          this.setAuthState({
            user: response.user,
            token: response.token,
            isAuthenticated: true,
          });
          // トークンを保存
          localStorage.setItem("auth_token", response.token);
          localStorage.setItem("user", JSON.stringify(response.user));
        }),
        catchError((error) => {
          console.error("Login failed", error);
          return of({ error: "Login failed" });
        })
      );
  }

  logout() {
    // 認証状態をリセット
    this.authState.set({
      user: null,
      token: null,
      isAuthenticated: false,
    });

    // ローカルストレージをクリア
    localStorage.removeItem("auth_token");
    localStorage.removeItem("user");

    // ログインページにリダイレクト
    this.router.navigate(["/login"]);
  }

  private setAuthState(state: AuthState) {
    this.authState.set(state);
  }

  private restoreAuthState() {
    const token = localStorage.getItem("auth_token");
    const userJson = localStorage.getItem("user");

    if (token && userJson) {
      try {
        const user = JSON.parse(userJson);
        this.setAuthState({
          user,
          token,
          isAuthenticated: true,
        });
      } catch (e) {
        console.error("Failed to restore auth state", e);
        this.logout();
      }
    }
  }

  // トークンの検証
  validateToken() {
    const currentToken = this.token();
    if (!currentToken) return of(false);

    return this.http
      .post<{ valid: boolean }>("/api/validate-token", { token: currentToken })
      .pipe(
        tap((response) => {
          if (!response.valid) {
            this.logout();
          }
        }),
        catchError(() => {
          this.logout();
          return of(false);
        })
      );
  }
}
```

## ガードでの使用例

```typescript
import { inject } from "@angular/core";
import { CanActivateFn, Router } from "@angular/router";
import { AuthService } from "./auth.service";

export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // 認証されていない場合はログインページにリダイレクト
  return router.parseUrl("/login");
};
```

## コンポーネントでの使用例

```typescript
import { Component } from "@angular/core";
import { AuthService } from "./auth.service";

@Component({
  selector: "app-header",
  template: `
    <nav>
      <div class="logo">MyApp</div>
      <div class="user-section" *ngIf="authService.isAuthenticated()">
        Welcome, {{ authService.user()?.name }}
        <button (click)="logout()">Logout</button>
      </div>
      <div class="login-section" *ngIf="!authService.isAuthenticated()">
        <a routerLink="/login">Login</a>
      </div>
    </nav>
  `,
})
export class HeaderComponent {
  constructor(public authService: AuthService) {}

  logout() {
    this.authService.logout();
  }
}
```

## メリット

1. **効率的な変更検出**：シグナルは変更があった場合のみ関連する部分を更新
2. **DI 互換性**：標準的な Angular の DI システムと簡単に統合
3. **テスト容易性**：モックやスタブを使ったテストが容易
4. **デバッグ性**：状態変化を明示的に追跡可能

## 注意点

1. 複雑な認証フローでは、より詳細な状態管理が必要になる場合がある
2. JWT トークンの有効期限や更新トークンの管理も考慮する必要がある

Signal-Based の認証サービスは、Angular の最新機能を活用することで、より効率的で保守しやすい認証システムを実現します。特に TypeScript との組み合わせにより、型安全な開発が可能になります。
