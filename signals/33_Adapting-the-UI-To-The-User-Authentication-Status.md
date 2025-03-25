# Adapting the UI To The User Authentication Status

ユーザー認証状態に基づいて UI を適応させることは、現代のウェブアプリケーションで重要な要素です。Angular のシグナルベースの認証システムを活用すると、この実装が効率的に行えます。

## 基本的なアプローチ

1. **条件付きレンダリング**

   - 認証状態に基づいて異なる UI コンポーネントを表示
   - `*ngIf`ディレクティブや三項演算子を使用

2. **ナビゲーション制御**

   - 認証状態に応じてナビゲーションメニューの項目を動的に表示/非表示
   - 特定のルートへのアクセスを制限

3. **コンテンツのパーソナライズ**
   - ログインユーザーの情報に基づいてコンテンツをカスタマイズ
   - ユーザーの権限レベルに応じた機能の表示

## 実装例

### コンポーネントテンプレートでの条件付きレンダリング

```html
<div class="app-container">
  <!-- 認証状態に基づくヘッダー表示 -->
  <header>
    <h1>My Application</h1>

    <!-- 非認証ユーザー向けコンテンツ -->
    @if (!authService.isAuthenticated()) {
    <div class="auth-actions">
      <button routerLink="/login">ログイン</button>
      <button routerLink="/register">登録</button>
    </div>
    }

    <!-- 認証済みユーザー向けコンテンツ -->
    @if (authService.isAuthenticated()) {
    <div class="user-panel">
      <span class="welcome-message"
        >ようこそ、{{ authService.user()?.name }}</span
      >
      <div class="dropdown">
        <button class="dropdown-toggle">アカウント</button>
        <div class="dropdown-menu">
          <a routerLink="/profile">プロフィール</a>
          <a routerLink="/settings">設定</a>
          <button (click)="logout()">ログアウト</button>
        </div>
      </div>
    </div>
    }
  </header>

  <!-- メインナビゲーション -->
  <nav>
    <ul>
      <li><a routerLink="/">ホーム</a></li>
      <li><a routerLink="/public">公開コンテンツ</a></li>

      <!-- 認証済みユーザーのみに表示 -->
      @if (authService.isAuthenticated()) {
      <li><a routerLink="/dashboard">ダッシュボード</a></li>
      <li><a routerLink="/my-content">マイコンテンツ</a></li>
      }

      <!-- 管理者のみに表示 -->
      @if (isAdmin()) {
      <li><a routerLink="/admin">管理パネル</a></li>
      }
    </ul>
  </nav>

  <!-- メインコンテンツ -->
  <main>
    <router-outlet></router-outlet>
  </main>
</div>
```

### コンポーネントクラス

```typescript
import { Component } from "@angular/core";
import { Router } from "@angular/router";
import { AuthService } from "../services/auth.service";

@Component({
  selector: "app-layout",
  templateUrl: "./layout.component.html",
  styleUrls: ["./layout.component.scss"],
})
export class LayoutComponent {
  constructor(public authService: AuthService, private router: Router) {}

  logout(): void {
    this.authService.logout();
    this.router.navigate(["/"]);
  }

  isAdmin(): boolean {
    const user = this.authService.user();
    return user?.roles?.includes("admin") || false;
  }
}
```

### スタンドアロンコンポーネントでの実装

```typescript
import { Component } from "@angular/core";
import { RouterLink, RouterOutlet } from "@angular/router";
import { NgIf } from "@angular/common";
import { AuthService } from "./services/auth.service";

@Component({
  selector: "app-navigation",
  standalone: true,
  imports: [RouterLink, RouterOutlet, NgIf],
  template: `
    <nav class="navbar">
      <a class="brand" routerLink="/">MyApp</a>

      <div class="nav-links">
        <a routerLink="/features">機能</a>
        <a routerLink="/pricing">料金</a>

        @if (authService.isAuthenticated()) {
        <a routerLink="/dashboard">ダッシュボード</a>
        <a routerLink="/account">アカウント</a>
        <button (click)="logout()" class="logout-btn">ログアウト</button>
        } @else {
        <a routerLink="/login" class="login-btn">ログイン</a>
        <a routerLink="/signup" class="signup-btn">登録</a>
        }
      </div>
    </nav>
  `,
})
export class NavigationComponent {
  constructor(public authService: AuthService) {}

  logout(): void {
    this.authService.logout();
  }
}
```

## 高度な実装パターン

### ユーザーロールに基づいた機能制御

```typescript
import { Component, computed } from "@angular/core";
import { AuthService } from "./auth.service";

@Component({
  selector: "app-feature-access",
  template: `
    <div class="feature-container">
      <!-- 基本機能: すべての認証ユーザー -->
      @if (authService.isAuthenticated()) {
      <div class="feature-block">
        <h3>基本機能</h3>
        <app-basic-features></app-basic-features>
      </div>
      }

      <!-- プレミアム機能: プレミアムユーザー -->
      @if (canAccessPremium()) {
      <div class="feature-block premium">
        <h3>プレミアム機能</h3>
        <app-premium-features></app-premium-features>
      </div>
      }

      <!-- 管理者機能: 管理者のみ -->
      @if (canAccessAdmin()) {
      <div class="feature-block admin">
        <h3>管理機能</h3>
        <app-admin-features></app-admin-features>
      </div>
      }

      <!-- 非認証ユーザー向け -->
      @if (!authService.isAuthenticated()) {
      <div class="login-prompt">
        <p>
          すべての機能にアクセスするには<a routerLink="/login">ログイン</a
          >してください
        </p>
      </div>
      }
    </div>
  `,
})
export class FeatureAccessComponent {
  constructor(public authService: AuthService) {}

  // 計算されたシグナル
  readonly canAccessPremium = computed(() => {
    const user = this.authService.user();
    return user?.subscription === "premium" || this.canAccessAdmin();
  });

  readonly canAccessAdmin = computed(() => {
    const user = this.authService.user();
    return user?.roles?.includes("admin") || false;
  });
}
```

### 動的な UI 状態の管理

```typescript
import { Injectable, signal, computed } from "@angular/core";
import { AuthService } from "./auth.service";

interface UiState {
  sidebarVisible: boolean;
  darkMode: boolean;
  compactView: boolean;
  notificationsEnabled: boolean;
}

@Injectable({
  providedIn: "root",
})
export class UiStateService {
  private state = signal<UiState>({
    sidebarVisible: true,
    darkMode: false,
    compactView: false,
    notificationsEnabled: true,
  });

  // 認証状態に応じたUIの設定
  constructor(private authService: AuthService) {
    // 認証状態の変化を監視
    effect(() => {
      if (this.authService.isAuthenticated()) {
        // ユーザー固有の設定を読み込む
        this.loadUserPreferences();
      } else {
        // 未認証状態ではデフォルト設定に戻す
        this.resetToDefaults();
      }
    });
  }

  // 公開シグナル
  readonly sidebar = computed(() => this.state().sidebarVisible);
  readonly darkMode = computed(() => this.state().darkMode);
  readonly compactView = computed(() => this.state().compactView);
  readonly notifications = computed(() => this.state().notificationsEnabled);

  // 設定変更メソッド
  toggleSidebar() {
    this.updateState({ sidebarVisible: !this.state().sidebarVisible });
    this.savePreferences();
  }

  toggleDarkMode() {
    this.updateState({ darkMode: !this.state().darkMode });
    this.savePreferences();
  }

  toggleCompactView() {
    this.updateState({ compactView: !this.state().compactView });
    this.savePreferences();
  }

  toggleNotifications() {
    this.updateState({
      notificationsEnabled: !this.state().notificationsEnabled,
    });
    this.savePreferences();
  }

  private updateState(partial: Partial<UiState>) {
    this.state.update((current) => ({ ...current, ...partial }));
  }

  private loadUserPreferences() {
    const userId = this.authService.user()?.id;
    if (!userId) return;

    const savedPrefs = localStorage.getItem(`ui_prefs_${userId}`);
    if (savedPrefs) {
      try {
        const prefs = JSON.parse(savedPrefs);
        this.updateState(prefs);
      } catch (e) {
        console.error("Error loading preferences", e);
      }
    }
  }

  private savePreferences() {
    if (!this.authService.isAuthenticated()) return;

    const userId = this.authService.user()?.id;
    if (userId) {
      localStorage.setItem(`ui_prefs_${userId}`, JSON.stringify(this.state()));
    }
  }

  private resetToDefaults() {
    this.state.set({
      sidebarVisible: true,
      darkMode: false,
      compactView: false,
      notificationsEnabled: true,
    });
  }
}
```

## ベストプラクティス

1. **レスポンシブな UI 設計**

   - 認証状態の変化にスムーズに対応する UI を設計
   - フェードイン/フェードアウトなどのトランジション効果を活用

2. **不要な画面のちらつき防止**

   - 初期ロード時に認証状態が確定するまでプレースホルダーを表示
   - `*ngIf` の代わりに `[hidden]` を使用して要素を視覚的に隠す場合も

3. **パーミッションベースのアプローチ**

   - ユーザーロールではなく具体的な権限に基づいて UI を制御
   - より柔軟で詳細な制御が可能

4. **プログレッシブエンハンスメント**
   - 基本機能をすべてのユーザーに提供し、認証ユーザーには機能を追加
   - 非認証ユーザーでも制限付きで使用できる機能を提供

Angular のシグナルベースの認証サービスを使用することで、認証状態の変化を効率的に追跡し、UI にリアルタイムで反映させることができます。これにより、ユーザーはスムーズな体験を得ることができ、開発者は保守しやすい清潔なコードを維持できます。
