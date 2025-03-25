# Signal-Based User Messages - Shared Service

## 概要

Signal-Based User Messages Shared Service は、Angular アプリケーション全体でユーザーへのメッセージ（通知、アラート、トースト）を一元管理するための仕組みです。Signals を活用することで、リアクティブかつ効率的なメッセージ管理が可能になります。

## 主要コンポーネント

1. **中央集権的なメッセージ管理**

   - アプリケーション全体のメッセージを単一のサービスで管理
   - 複数のコンポーネントからアクセス可能

2. **Signal API を活用した状態管理**

   - `signal()`, `computed()`, `effect()`を使ったメッセージ状態管理
   - リアクティブなメッセージの追加・削除・更新

3. **メッセージタイプと優先度**
   - 情報、成功、警告、エラーなどの異なるメッセージタイプ
   - メッセージの重要度に基づいた表示制御

## 実装例

### メッセージインターフェイスの定義

```typescript
// message.model.ts
export type MessageType = "info" | "success" | "warning" | "error";
export type MessagePriority = "low" | "medium" | "high";

export interface UserMessage {
  id: string; // 一意のID
  type: MessageType; // メッセージタイプ
  text: string; // メッセージテキスト
  title?: string; // オプションのタイトル
  priority: MessagePriority; // 優先度
  autoClose?: boolean; // 自動で閉じるか
  duration?: number; // 表示時間（ミリ秒）
  timestamp: number; // 作成時刻
  data?: any; // 追加データ
  action?: {
    // オプションのアクション
    text: string;
    callback: () => void;
  };
}
```

### Shared Message Service

```typescript
// user-messages.service.ts
import { Injectable, signal, computed, effect } from "@angular/core";
import { UserMessage, MessageType, MessagePriority } from "./message.model";

@Injectable({
  providedIn: "root",
})
export class UserMessagesService {
  // メッセージのシグナル
  private messages = signal<UserMessage[]>([]);

  // 派生したcomputed values
  public activeMessages = computed(() => this.messages());
  public hasMessages = computed(() => this.messages().length > 0);
  public errorCount = computed(
    () => this.messages().filter((msg) => msg.type === "error").length
  );

  // 最新のメッセージ
  public latestMessage = computed(() => {
    const msgs = this.messages();
    return msgs.length ? msgs[msgs.length - 1] : null;
  });

  constructor() {
    // エラーメッセージがある場合のeffect
    effect(() => {
      const errorCount = this.errorCount();
      if (errorCount > 0) {
        console.warn(`${errorCount}件のエラーメッセージがあります`);
      }
    });

    // 自動クローズタイマーの設定
    effect(() => {
      const messages = this.messages();
      messages.forEach((message) => {
        if (message.autoClose && !message._timeoutId) {
          this.setupAutoClose(message);
        }
      });
    });
  }

  // 新しいメッセージを追加
  addMessage(
    text: string,
    type: MessageType = "info",
    options: {
      title?: string;
      priority?: MessagePriority;
      autoClose?: boolean;
      duration?: number;
      data?: any;
      action?: { text: string; callback: () => void };
    } = {}
  ): string {
    const id = `msg-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

    const message: UserMessage = {
      id,
      type,
      text,
      title: options.title,
      priority: options.priority || "medium",
      autoClose: options.autoClose !== undefined ? options.autoClose : true,
      duration: options.duration || this.getDefaultDuration(type),
      timestamp: Date.now(),
      data: options.data,
      action: options.action,
      _timeoutId: undefined,
    };

    this.messages.update((msgs) => [...msgs, message]);

    return id;
  }

  // 成功メッセージの追加
  addSuccess(text: string, options = {}): string {
    return this.addMessage(text, "success", options);
  }

  // 警告メッセージの追加
  addWarning(text: string, options = {}): string {
    return this.addMessage(text, "warning", options);
  }

  // エラーメッセージの追加
  addError(text: string, options = {}): string {
    return this.addMessage(text, "error", {
      autoClose: false,
      priority: "high",
      ...options,
    });
  }

  // メッセージの削除
  removeMessage(id: string): void {
    const message = this.messages().find((m) => m.id === id);

    if (message && message._timeoutId) {
      clearTimeout(message._timeoutId);
    }

    this.messages.update((msgs) => msgs.filter((msg) => msg.id !== id));
  }

  // すべてのメッセージをクリア
  clearAll(): void {
    // すべての自動クローズタイマーをクリア
    this.messages().forEach((message) => {
      if (message._timeoutId) {
        clearTimeout(message._timeoutId);
      }
    });

    this.messages.set([]);
  }

  // 特定のタイプのメッセージをすべてクリア
  clearByType(type: MessageType): void {
    const messagesToRemove = this.messages().filter((msg) => msg.type === type);

    // タイマーをクリア
    messagesToRemove.forEach((message) => {
      if (message._timeoutId) {
        clearTimeout(message._timeoutId);
      }
    });

    this.messages.update((msgs) => msgs.filter((msg) => msg.type !== type));
  }

  // メッセージタイプに基づくデフォルト表示時間
  private getDefaultDuration(type: MessageType): number {
    switch (type) {
      case "error":
        return 0; // エラーは自動で閉じない
      case "warning":
        return 5000; // 5秒
      case "success":
        return 3000; // 3秒
      case "info":
      default:
        return 3000; // 3秒
    }
  }

  // 自動クローズタイマーのセットアップ
  private setupAutoClose(message: UserMessage): void {
    if (!message.autoClose || message.duration === 0) return;

    const timeoutId = setTimeout(() => {
      this.removeMessage(message.id);
    }, message.duration);

    // タイムアウトIDをメッセージに保存
    message._timeoutId = timeoutId;
  }
}
```

### メッセージ表示コンポーネント

```typescript
// messages-container.component.ts
import { Component, inject } from "@angular/core";
import { UserMessagesService } from "./user-messages.service";
import { UserMessage } from "./message.model";

@Component({
  selector: "app-messages-container",
  template: `
    <div class="messages-container" *ngIf="messagesService.hasMessages()">
      <div
        *ngFor="let message of messagesService.activeMessages()"
        class="message-item"
        [ngClass]="'message-' + message.type"
      >
        <div class="message-header">
          <span class="message-title">{{
            message.title || getDefaultTitle(message.type)
          }}</span>
          <button class="close-btn" (click)="closeMessage(message.id)">
            ×
          </button>
        </div>

        <div class="message-body">
          {{ message.text }}
        </div>

        <div class="message-actions" *ngIf="message.action">
          <button (click)="handleAction(message)">
            {{ message.action.text }}
          </button>
        </div>
      </div>
    </div>
  `,
  styles: [
    `
      .messages-container {
        position: fixed;
        top: 20px;
        right: 20px;
        z-index: 1000;
        width: 300px;
      }

      .message-item {
        margin-bottom: 10px;
        padding: 15px;
        border-radius: 4px;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
        animation: slideIn 0.3s ease-out;
      }

      .message-info {
        background-color: #f0f7ff;
        border-left: 4px solid #2196f3;
      }

      .message-success {
        background-color: #e8f5e9;
        border-left: 4px solid #4caf50;
      }

      .message-warning {
        background-color: #fff8e1;
        border-left: 4px solid #ff9800;
      }

      .message-error {
        background-color: #ffebee;
        border-left: 4px solid #f44336;
      }

      @keyframes slideIn {
        from {
          transform: translateX(100%);
          opacity: 0;
        }
        to {
          transform: translateX(0);
          opacity: 1;
        }
      }
    `,
  ],
})
export class MessagesContainerComponent {
  messagesService = inject(UserMessagesService);

  getDefaultTitle(type: string): string {
    switch (type) {
      case "info":
        return "情報";
      case "success":
        return "成功";
      case "warning":
        return "注意";
      case "error":
        return "エラー";
      default:
        return "";
    }
  }

  closeMessage(id: string): void {
    this.messagesService.removeMessage(id);
  }

  handleAction(message: UserMessage): void {
    if (message.action && message.action.callback) {
      message.action.callback();
    }

    // アクション実行後にメッセージを閉じる（オプション）
    this.messagesService.removeMessage(message.id);
  }
}
```

## 使用例

### コンポーネントでの使用

```typescript
// course.component.ts
import { Component, inject } from "@angular/core";
import { UserMessagesService } from "./user-messages.service";
import { CourseService } from "./course.service";

@Component({
  selector: "app-course",
  template: `
    <div class="course-container">
      <h2>コース詳細</h2>

      <button (click)="enrollCourse()">コースに登録</button>
    </div>
  `,
})
export class CourseComponent {
  private messagesService = inject(UserMessagesService);
  private courseService = inject(CourseService);

  enrollCourse(): void {
    this.courseService.enroll("angular-signals-101").subscribe({
      next: () => {
        this.messagesService.addSuccess("コースへの登録が完了しました！", {
          title: "登録成功",
          action: {
            text: "コースを開始",
            callback: () => this.startCourse(),
          },
        });
      },
      error: (err) => {
        this.messagesService.addError(
          `コース登録中にエラーが発生しました: ${err.message}`,
          { title: "登録エラー" }
        );
      },
    });
  }

  startCourse(): void {
    // コース開始ロジック
    console.log("コースを開始します");
  }
}
```

### 複数メッセージの表示と順序管理

```typescript
// registration.component.ts
import { Component, inject } from "@angular/core";
import { UserMessagesService } from "./user-messages.service";

@Component({
  selector: "app-registration",
  template: `
    <div class="registration-form">
      <!-- フォーム内容 -->

      <button (click)="completeRegistration()">登録完了</button>
    </div>
  `,
})
export class RegistrationComponent {
  private messagesService = inject(UserMessagesService);

  completeRegistration(): void {
    // 複数のメッセージを表示

    // まず成功メッセージ
    this.messagesService.addSuccess("アカウント登録が完了しました！");

    // 次に情報メッセージ
    this.messagesService.addMessage(
      "メールアドレスの確認のため、確認メールを送信しました。",
      "info",
      { duration: 8000 }
    );

    // 最後に警告メッセージ
    this.messagesService.addWarning(
      "プロフィール情報が未完成です。プロフィールページで情報を更新してください。",
      {
        action: {
          text: "プロフィールへ",
          callback: () => this.navigateToProfile(),
        },
      }
    );
  }

  navigateToProfile(): void {
    // プロフィールページへの遷移
    console.log("プロフィールページへ遷移");
  }
}
```

### HTTP Interceptor と Signal-Based Messages の連携

```typescript
// http-error.interceptor.ts
import { Injectable, inject } from "@angular/core";
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse,
} from "@angular/common/http";
import { Observable, catchError, throwError } from "rxjs";
import { UserMessagesService } from "./user-messages.service";

@Injectable()
export class HttpErrorInterceptor implements HttpInterceptor {
  private messagesService = inject(UserMessagesService);

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      catchError((error: HttpErrorResponse) => {
        let errorMessage = "";

        // エラータイプに基づいてメッセージを生成
        if (error.error instanceof ErrorEvent) {
          // クライアントサイドエラー
          errorMessage = `クライアントエラー: ${error.error.message}`;
        } else {
          // サーバーサイドエラー
          switch (error.status) {
            case 401:
              errorMessage = "認証に失敗しました。再度ログインしてください。";
              break;
            case 403:
              errorMessage = "このリソースにアクセスする権限がありません。";
              break;
            case 404:
              errorMessage = "要求されたリソースが見つかりませんでした。";
              break;
            case 500:
              errorMessage =
                "サーバー内部エラーが発生しました。後ほど再試行してください。";
              break;
            default:
              errorMessage = `サーバーエラー: ${error.status} - ${error.message}`;
          }
        }

        // エラーメッセージを表示
        this.messagesService.addError(errorMessage, {
          data: error, // デバッグ情報としてエラーオブジェクトを含める
        });

        // エラーを次のハンドラーに渡す
        return throwError(() => error);
      })
    );
  }
}
```

## 高度な機能

### メッセージの優先順位付け

優先度に基づいたメッセージの表示順序設定：

```typescript
// 優先度に基づいて並べ替えたメッセージ
public sortedMessages = computed(() => {
  const priorityOrder = { high: 0, medium: 1, low: 2 };

  return [...this.messages()].sort((a, b) => {
    // まず優先度で並べ替え
    const priorityDiff = priorityOrder[a.priority] - priorityOrder[b.priority];
    if (priorityDiff !== 0) return priorityDiff;

    // 同じ優先度なら新しいものが上に来るように
    return b.timestamp - a.timestamp;
  });
});
```

### メッセージのグループ化

関連するメッセージをグループ化して表示：

```typescript
// messages-container.component.ts
// メッセージをタイプごとにグループ化
get groupedMessages() {
  const groups = {
    error: [],
    warning: [],
    success: [],
    info: []
  };

  this.messagesService.activeMessages().forEach(msg => {
    groups[msg.type].push(msg);
  });

  return groups;
}
```

### メッセージの重複防止

同じ内容のメッセージの重複を防止：

```typescript
// user-messages.service.ts
addMessage(text: string, type: MessageType = 'info', options = {}): string {
  // 既に同じメッセージが存在するか確認
  const existingMessage = this.messages().find(msg =>
    msg.text === text && msg.type === type
  );

  if (existingMessage) {
    // 既存のメッセージをリセット（タイマーをリセットするなど）
    this.refreshMessage(existingMessage.id);
    return existingMessage.id;
  }

  // 新しいメッセージを追加
  const id = `msg-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  // ...
}

// メッセージをリフレッシュ（タイマーのリセットなど）
refreshMessage(id: string): void {
  this.messages.update(msgs => {
    return msgs.map(msg => {
      if (msg.id === id) {
        // タイマーをクリア
        if (msg._timeoutId) {
          clearTimeout(msg._timeoutId);
          msg._timeoutId = undefined;
        }

        // タイムスタンプを更新
        return { ...msg, timestamp: Date.now() };
      }
      return msg;
    });
  });
}
```

## メリット

1. **一貫したユーザーフィードバック** - アプリケーション全体で統一されたメッセージ表示
2. **リアクティブな更新** - Signals による効率的な UI 更新
3. **柔軟なカスタマイズ** - 様々なメッセージタイプと表示オプション
4. **開発者体験の向上** - 簡単で直感的な API
5. **デバッグのしやすさ** - 中央集権的なメッセージ管理

## ベストプラクティス

1. **メッセージの階層化** - 重要度や関連性に基づいたメッセージのグループ化
2. **コンテキスト提供** - エラーメッセージには解決策やアクションを含める
3. **アクセシビリティ考慮** - スクリーンリーダー対応や色だけに依存しない識別方法
4. **メッセージの寿命管理** - 適切なタイミングでのメッセージ消去
5. **ユーザーコントロール** - ユーザーがメッセージを閉じる/無視できる機能

Signal-Based User Messages Shared Service を活用することで、リアクティブでユーザーフレンドリーなフィードバックシステムを実現できます。これにより、ユーザーエクスペリエンスの向上とアプリケーションの使いやすさが大幅に改善されます。
