# Save User Profile In Local Storage - Survive Refreshes

ユーザープロファイルをローカルストレージに保存することで、ページの更新やブラウザの再起動後も認証状態を維持できます。この方法により、ユーザーは毎回ログインする必要がなくなり、シームレスなユーザー体験を提供できます。

## 基本的な実装方法

### 1. ユーザープロファイルの保存

```typescript
import { Injectable, signal, computed } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Router } from "@angular/router";
import { tap } from "rxjs/operators";

export interface UserProfile {
  id: string;
  email: string;
  name: string;
  avatar?: string;
  roles?: string[];
  preferences?: Record<string, any>;
}

@Injectable({
  providedIn: "root",
})
export class AuthService {
  // ユーザープロファイルをシグナルとして定義
  private userProfileSignal = signal<UserProfile | null>(null);
  // 認証状態のシグナル
  private isAuthenticatedSignal = signal<boolean>(false);

  // 公開シグナル
  readonly userProfile = computed(() => this.userProfileSignal());
  readonly isAuthenticated = computed(() => this.isAuthenticatedSignal());

  private readonly STORAGE_KEY = "user_profile";
  private readonly TOKEN_KEY = "auth_token";

  constructor(private http: HttpClient, private router: Router) {
    // 初期化時にローカルストレージから認証情報を復元
    this.restoreAuthState();
  }

  login(email: string, password: string) {
    return this.http
      .post<{ user: UserProfile; token: string }>("/api/login", {
        email,
        password,
      })
      .pipe(
        tap((response) => {
          // ユーザープロファイルをシグナルに設定
          this.userProfileSignal.set(response.user);
          this.isAuthenticatedSignal.set(true);

          // ローカルストレージに保存
          this.saveToLocalStorage(response.user, response.token);
        })
      );
  }

  logout() {
    // シグナルをリセット
    this.userProfileSignal.set(null);
    this.isAuthenticatedSignal.set(false);

    // ローカルストレージをクリア
    this.clearLocalStorage();

    // ログインページへリダイレクト
    this.router.navigate(["/login"]);
  }

  // ローカルストレージにユーザープロファイルとトークンを保存
  private saveToLocalStorage(user: UserProfile, token: string): void {
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(user));
    localStorage.setItem(this.TOKEN_KEY, token);
  }

  // ローカルストレージから認証情報を復元
  private restoreAuthState(): void {
    try {
      const storedProfile = localStorage.getItem(this.STORAGE_KEY);
      const storedToken = localStorage.getItem(this.TOKEN_KEY);

      if (storedProfile && storedToken) {
        const userProfile = JSON.parse(storedProfile) as UserProfile;
        this.userProfileSignal.set(userProfile);
        this.isAuthenticatedSignal.set(true);

        // オプション: トークンの有効性をバックエンドで検証
        this.validateToken(storedToken);
      }
    } catch (error) {
      console.error("Failed to restore auth state:", error);
      this.clearLocalStorage();
    }
  }

  // ローカルストレージをクリア
  private clearLocalStorage(): void {
    localStorage.removeItem(this.STORAGE_KEY);
    localStorage.removeItem(this.TOKEN_KEY);
  }

  // トークンの有効性を検証
  private validateToken(token: string): void {
    this.http
      .post<{ valid: boolean }>("/api/validate-token", { token })
      .subscribe({
        next: (response) => {
          if (!response.valid) {
            this.logout();
          }
        },
        error: () => {
          this.logout();
        },
      });
  }

  // ユーザープロファイルの更新
  updateProfile(updates: Partial<UserProfile>) {
    const currentProfile = this.userProfileSignal();
    if (!currentProfile) return;

    return this.http
      .patch<UserProfile>(`/api/users/${currentProfile.id}`, updates)
      .pipe(
        tap((updatedProfile) => {
          // 更新されたプロファイルをシグナルとローカルストレージに保存
          this.userProfileSignal.set(updatedProfile);

          // トークンは変更しないので、現在のトークンを取得
          const currentToken = localStorage.getItem(this.TOKEN_KEY) || "";

          // 更新したプロファイルをローカルストレージに保存
          this.saveToLocalStorage(updatedProfile, currentToken);
        })
      );
  }
}
```

### 2. ユーザープリファレンスの管理

```typescript
import { Injectable, inject } from "@angular/core";
import { AuthService, UserProfile } from "./auth.service";

interface UserPreferences {
  theme: "light" | "dark" | "system";
  language: string;
  notifications: boolean;
  compactView: boolean;
}

@Injectable({
  providedIn: "root",
})
export class PreferencesService {
  private authService = inject(AuthService);

  // デフォルトの設定
  private readonly DEFAULT_PREFERENCES: UserPreferences = {
    theme: "system",
    language: "ja",
    notifications: true,
    compactView: false,
  };

  // 現在のユーザー設定を取得
  getCurrentPreferences(): UserPreferences {
    const userProfile = this.authService.userProfile();
    return (
      (userProfile?.preferences as UserPreferences) || this.DEFAULT_PREFERENCES
    );
  }

  // 特定の設定を更新
  updatePreference<K extends keyof UserPreferences>(
    key: K,
    value: UserPreferences[K]
  ) {
    const userProfile = this.authService.userProfile();
    if (!userProfile) return;

    const currentPrefs = userProfile.preferences || {};
    const updatedPrefs = { ...currentPrefs, [key]: value };

    this.authService
      .updateProfile({
        preferences: updatedPrefs,
      })
      .subscribe();

    // 即時UIに反映させるために設定を適用
    this.applyPreference(key, value);
  }

  // 設定をUIに適用
  private applyPreference<K extends keyof UserPreferences>(
    key: K,
    value: UserPreferences[K]
  ) {
    switch (key) {
      case "theme":
        document.documentElement.setAttribute("data-theme", value as string);
        break;
      case "language":
        // 言語設定を適用するロジック
        break;
      // 他の設定に対する処理
    }
  }

  // すべての設定を初期化
  applyAllPreferences() {
    const prefs = this.getCurrentPreferences();

    // 各設定を適用
    Object.entries(prefs).forEach(([key, value]) => {
      this.applyPreference(key as keyof UserPreferences, value);
    });
  }
}
```

### 3. アプリ起動時の認証状態復元

```typescript
import { Component, OnInit } from "@angular/core";
import { AuthService } from "./services/auth.service";
import { PreferencesService } from "./services/preferences.service";

@Component({
  selector: "app-root",
  template: `
    <div class="app-container" [ngClass]="{ loading: loading }">
      <app-loading *ngIf="loading"></app-loading>

      <ng-container *ngIf="!loading">
        <app-header></app-header>
        <main>
          <router-outlet></router-outlet>
        </main>
        <app-footer></app-footer>
      </ng-container>
    </div>
  `,
})
export class AppComponent implements OnInit {
  loading = true;

  constructor(
    private authService: AuthService,
    private preferencesService: PreferencesService
  ) {}

  ngOnInit() {
    // アプリ起動時の初期化
    this.initializeApp();
  }

  private async initializeApp() {
    try {
      // 認証状態は AuthService のコンストラクタで自動的に復元される

      // ユーザー設定を適用
      if (this.authService.isAuthenticated()) {
        this.preferencesService.applyAllPreferences();
      }
    } catch (error) {
      console.error("App initialization error:", error);
    } finally {
      // 初期化が完了したらローディング状態を解除
      setTimeout(() => {
        this.loading = false;
      }, 500); // 最低限のローディング表示時間を確保
    }
  }
}
```

## セキュリティ上の考慮事項

### 1. センシティブ情報の保護

```typescript
// ユーザープロファイルから保存する前に機密情報を除外
private sanitizeProfileForStorage(profile: UserProfile): Partial<UserProfile> {
  // スプレッド構文で新しいオブジェクトを作成
  const { /* 除外するプロパティ */ passwordHash, securityQuestion, ...safeProfile } = profile;
  return safeProfile;
}
```

### 2. トークンの有効期限管理

```typescript
interface TokenData {
  token: string;
  expiresAt: number; // UNIXタイムスタンプ
}

// トークンをタイムスタンプ付きで保存
private saveToken(token: string, expiresIn: number): void {
  const expiresAt = Date.now() + expiresIn * 1000;
  const tokenData: TokenData = { token, expiresAt };
  localStorage.setItem(this.TOKEN_KEY, JSON.stringify(tokenData));
}

// トークンが有効期限内かチェック
private isTokenValid(): boolean {
  const tokenDataStr = localStorage.getItem(this.TOKEN_KEY);
  if (!tokenDataStr) return false;

  try {
    const tokenData: TokenData = JSON.parse(tokenDataStr);
    return Date.now() < tokenData.expiresAt;
  } catch {
    return false;
  }
}
```

### 3. リフレッシュトークンの実装

```typescript
interface AuthTokens {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

// アクセストークンの更新
refreshToken() {
  const refreshToken = this.getRefreshToken();
  if (!refreshToken) {
    this.logout();
    return;
  }

  return this.http.post<AuthTokens>('/api/refresh-token', { refreshToken })
    .pipe(
      tap(tokens => {
        this.saveTokens(tokens);
      }),
      catchError(error => {
        this.logout();
        return throwError(() => error);
      })
    );
}

// トークンの自動更新設定
setupTokenRefresh() {
  const tokenData = this.getTokenData();
  if (!tokenData) return;

  // 有効期限の5分前に更新
  const timeToRefresh = tokenData.expiresAt - Date.now() - (5 * 60 * 1000);
  if (timeToRefresh <= 0) {
    // すでに期限切れか期限間近の場合は即時更新
    this.refreshToken().subscribe();
  } else {
    // 期限切れ前にトークンを更新するためのタイマーを設定
    setTimeout(() => {
      this.refreshToken().subscribe();
    }, timeToRefresh);
  }
}
```

## ベストプラクティス

1. **必要な情報のみを保存する**

   - パスワードやセキュリティ情報などの機密データは絶対に保存しない
   - UI 表示に必要なデータのみをローカルストレージに保存

2. **データの暗号化**

   - 特に重要な情報は暗号化してから保存する
   - サードパーティライブラリ（CryptoJS 等）を使用して実装

3. **自動ログアウト**

   - 長時間の非アクティブ状態を検出して自動ログアウト
   - `sessionStorage`の使用も検討（ブラウザを閉じると消去される）

4. **バックエンド検証**

   - ローカルストレージのデータだけでなく、常にバックエンドでトークンを検証
   - エラー時は自動的にログアウト処理を実行

5. **マルチタブ対応**
   - 複数タブでのログイン/ログアウト状態を同期
   - `localStorage`イベントを使用して同期を実装

```typescript
// 他のタブでのストレージ変更を検知
window.addEventListener("storage", (event) => {
  if (event.key === this.TOKEN_KEY && !event.newValue) {
    // 他のタブでログアウトした場合
    this.userProfileSignal.set(null);
    this.isAuthenticatedSignal.set(false);
    this.router.navigate(["/login"]);
  } else if (event.key === this.TOKEN_KEY && event.newValue) {
    // 他のタブでログインした場合
    this.restoreAuthState();
  }
});
```

## まとめ

ローカルストレージを使ってユーザープロファイルを保存することで、ページ更新後も認証状態を維持でき、ユーザー体験を大幅に向上させることができます。ただし、セキュリティ上の考慮事項を無視せず、適切な対策を施すことが重要です。特に機密情報の保護、トークンの有効期限管理、マルチタブ対応などを実装することで、より堅牢な認証システムを構築できます。

Angular のシグナルベースのアプローチを使用すると、状態管理が簡素化され、リアクティブな UI 更新が可能になります。これにより、認証状態の変化にスムーズに対応し、ユーザーにシームレスな体験を提供できます。
