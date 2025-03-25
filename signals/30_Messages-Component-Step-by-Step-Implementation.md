# Messages Component - Step-by-Step Implementation

## 概要

メッセージコンポーネントの実装手順を段階的に解説します。このコンポーネントは Angular Signals を活用して、アプリケーション全体でユーザーメッセージを表示・管理します。

## 実装手順

### Step 1: メッセージモデルの定義

```typescript
// models/message.model.ts
export type MessageType = "info" | "success" | "warning" | "error";

export interface Message {
  id: string;
  text: string;
  type: MessageType;
  autoClose: boolean;
  duration?: number;
  timestamp: number;
}
```

### Step 2: メッセージサービスの作成

```typescript
// services/message.service.ts
import { Injectable, signal, computed } from "@angular/core";
import { Message, MessageType } from "../models/message.model";

@Injectable({
  providedIn: "root",
})
export class MessageService {
  // 基本的なメッセージ保持用のシグナル
  private messagesSignal = signal<Message[]>([]);

  // パブリックな読み取り専用シグナル
  public messages = computed(() => this.messagesSignal());

  // メッセージの追加
  addMessage(
    text: string,
    type: MessageType = "info",
    autoClose = true,
    duration?: number
  ): string {
    const id = Date.now().toString();

    const message: Message = {
      id,
      text,
      type,
      autoClose,
      duration: duration || this.getDurationByType(type),
      timestamp: Date.now(),
    };

    this.messagesSignal.update((messages) => [...messages, message]);

    // 自動クローズの設定
    if (autoClose && duration !== 0) {
      setTimeout(() => {
        this.removeMessage(id);
      }, message.duration);
    }

    return id;
  }

  // ショートカットメソッド
  addInfo(text: string, autoClose = true): string {
    return this.addMessage(text, "info", autoClose);
  }

  addSuccess(text: string, autoClose = true): string {
    return this.addMessage(text, "success", autoClose);
  }

  addWarning(text: string, autoClose = true): string {
    return this.addMessage(text, "warning", autoClose);
  }

  addError(text: string, autoClose = false): string {
    return this.addMessage(text, "error", autoClose);
  }

  // メッセージの削除
  removeMessage(id: string): void {
    this.messagesSignal.update((messages) =>
      messages.filter((message) => message.id !== id)
    );
  }

  // すべてのメッセージをクリア
  clearAll(): void {
    this.messagesSignal.set([]);
  }

  // タイプごとのデフォルト表示時間を取得
  private getDurationByType(type: MessageType): number {
    switch (type) {
      case "info":
        return 3000;
      case "success":
        return 3000;
      case "warning":
        return 5000;
      case "error":
        return 0; // エラーは自動で閉じない
      default:
        return 3000;
    }
  }
}
```

### Step 3: メッセージコンテナコンポーネントの作成

```typescript
// components/message-container/message-container.component.ts
import { Component, inject } from "@angular/core";
import { CommonModule } from "@angular/common";
import { MessageService } from "../../services/message.service";
import { MessageItemComponent } from "../message-item/message-item.component";

@Component({
  selector: "app-message-container",
  standalone: true,
  imports: [CommonModule, MessageItemComponent],
  template: `
    <div class="message-container" *ngIf="messageService.messages().length > 0">
      <app-message-item
        *ngFor="let message of messageService.messages()"
        [message]="message"
        (close)="closeMessage($event)"
      ></app-message-item>
    </div>
  `,
  styles: [
    `
      .message-container {
        position: fixed;
        top: 20px;
        right: 20px;
        max-width: 350px;
        z-index: 1000;
      }
    `,
  ],
})
export class MessageContainerComponent {
  messageService = inject(MessageService);

  closeMessage(id: string): void {
    this.messageService.removeMessage(id);
  }
}
```

### Step 4: メッセージアイテムコンポーネントの作成

```typescript
// components/message-item/message-item.component.ts
import { Component, Input, Output, EventEmitter } from "@angular/core";
import { CommonModule } from "@angular/common";
import { Message } from "../../models/message.model";

@Component({
  selector: "app-message-item",
  standalone: true,
  imports: [CommonModule],
  template: `
    <div
      class="message-item"
      [ngClass]="'message-' + message.type"
      [@fadeInOut]
    >
      <div class="message-content">
        <div class="message-icon" [ngClass]="'icon-' + message.type">
          <i [ngClass]="getIconClass()"></i>
        </div>
        <div class="message-text">{{ message.text }}</div>
      </div>
      <button class="close-button" (click)="onClose()">×</button>
    </div>
  `,
  styles: [
    `
      .message-item {
        display: flex;
        justify-content: space-between;
        align-items: flex-start;
        margin-bottom: 10px;
        padding: 12px 15px;
        border-radius: 4px;
        box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        background-color: white;
        max-width: 100%;
      }

      .message-content {
        display: flex;
        align-items: flex-start;
        flex: 1;
      }

      .message-icon {
        margin-right: 12px;
      }

      .message-info {
        border-left: 4px solid #2196f3;
      }

      .message-success {
        border-left: 4px solid #4caf50;
      }

      .message-warning {
        border-left: 4px solid #ff9800;
      }

      .message-error {
        border-left: 4px solid #f44336;
      }

      .icon-info {
        color: #2196f3;
      }
      .icon-success {
        color: #4caf50;
      }
      .icon-warning {
        color: #ff9800;
      }
      .icon-error {
        color: #f44336;
      }

      .close-button {
        background: none;
        border: none;
        font-size: 20px;
        cursor: pointer;
        opacity: 0.5;
      }

      .close-button:hover {
        opacity: 1;
      }
    `,
  ],
  animations: [
    // アニメーション定義
  ],
})
export class MessageItemComponent {
  @Input() message!: Message;
  @Output() close = new EventEmitter<string>();

  onClose(): void {
    this.close.emit(this.message.id);
  }

  getIconClass(): string {
    switch (this.message.type) {
      case "info":
        return "bi bi-info-circle-fill";
      case "success":
        return "bi bi-check-circle-fill";
      case "warning":
        return "bi bi-exclamation-triangle-fill";
      case "error":
        return "bi bi-x-circle-fill";
      default:
        return "bi bi-bell-fill";
    }
  }
}
```

### Step 5: アニメーションの追加

```typescript
// components/message-item/message-item.component.ts
import { trigger, style, animate, transition } from "@angular/animations";

// コンポーネントデコレータ内のanimations部分
animations: [
  trigger("fadeInOut", [
    transition(":enter", [
      style({ opacity: 0, transform: "translateX(40px)" }),
      animate(
        "300ms ease-out",
        style({ opacity: 1, transform: "translateX(0)" })
      ),
    ]),
    transition(":leave", [
      animate(
        "200ms ease-in",
        style({ opacity: 0, transform: "translateX(40px)" })
      ),
    ]),
  ]),
];
```

### Step 6: メインアプリコンポーネントへの統合

```typescript
// app.component.ts
import { Component } from "@angular/core";
import { MessageContainerComponent } from "./components/message-container/message-container.component";

@Component({
  selector: "app-root",
  imports: [MessageContainerComponent],
  template: `
    <div class="app-container">
      <!-- アプリのメインコンテンツ -->
      <router-outlet></router-outlet>

      <!-- グローバルメッセージコンテナ -->
      <app-message-container></app-message-container>
    </div>
  `,
})
export class AppComponent {}
```

### Step 7: サービスの機能拡張 - メッセージのグループ化

```typescript
// services/message.service.ts - 追加メソッド
// タイプごとにグループ化されたメッセージを取得
public groupedMessages = computed(() => {
  const msgs = this.messagesSignal();
  const grouped = {
    info: msgs.filter(m => m.type === 'info'),
    success: msgs.filter(m => m.type === 'success'),
    warning: msgs.filter(m => m.type === 'warning'),
    error: msgs.filter(m => m.type === 'error')
  };

  return grouped;
});

// 特定のタイプのメッセージ数を取得
public errorCount = computed(() =>
  this.messagesSignal().filter(m => m.type === 'error').length
);

public warningCount = computed(() =>
  this.messagesSignal().filter(m => m.type === 'warning').length
);
```

### Step 8: 特定のアクションを含むメッセージの実装

```typescript
// models/message.model.ts - 拡張
export interface Message {
  id: string;
  text: string;
  type: MessageType;
  autoClose: boolean;
  duration?: number;
  timestamp: number;
  action?: {
    text: string;
    handler: () => void;
  };
}

// services/message.service.ts - メソッド拡張
addMessage(
  text: string,
  type: MessageType = 'info',
  options: {
    autoClose?: boolean;
    duration?: number;
    action?: { text: string; handler: () => void };
  } = {}
): string {
  const id = Date.now().toString();

  const message: Message = {
    id,
    text,
    type,
    autoClose: options.autoClose !== undefined ? options.autoClose : true,
    duration: options.duration || this.getDurationByType(type),
    timestamp: Date.now(),
    action: options.action
  };

  this.messagesSignal.update(messages => [...messages, message]);

  if (message.autoClose && message.duration !== 0) {
    setTimeout(() => {
      this.removeMessage(id);
    }, message.duration);
  }

  return id;
}
```

### Step 9: メッセージアイテムコンポーネントのアクション対応

```typescript
// components/message-item/message-item.component.ts - テンプレート更新
template: `
  <div class="message-item" [ngClass]="'message-' + message.type" [@fadeInOut]>
    <div class="message-content">
      <div class="message-icon" [ngClass]="'icon-' + message.type">
        <i [ngClass]="getIconClass()"></i>
      </div>
      <div class="message-text">
        {{ message.text }}

        <div class="message-action" *ngIf="message.action">
          <button
            class="action-button"
            (click)="onActionClick()"
          >
            {{ message.action.text }}
          </button>
        </div>
      </div>
    </div>
    <button class="close-button" (click)="onClose()">×</button>
  </div>
`

// コンポーネントクラスに追加
onActionClick(): void {
  if (this.message.action && this.message.action.handler) {
    this.message.action.handler();
    // アクション実行後にメッセージを閉じる
    this.close.emit(this.message.id);
  }
}
```

### Step 10: HTTP エラーとの統合

```typescript
// http-error.interceptor.ts
import { Injectable } from "@angular/core";
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse,
} from "@angular/common/http";
import { Observable, throwError } from "rxjs";
import { catchError } from "rxjs/operators";
import { MessageService } from "../services/message.service";

@Injectable()
export class HttpErrorInterceptor implements HttpInterceptor {
  constructor(private messageService: MessageService) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      catchError((error: HttpErrorResponse) => {
        let errorMessage = "";

        if (error.error instanceof ErrorEvent) {
          // クライアントサイドエラー
          errorMessage = `クライアントエラー: ${error.error.message}`;
        } else {
          // サーバーサイドエラー
          switch (error.status) {
            case 401:
              errorMessage = "認証エラー: ログインしてください";
              break;
            case 403:
              errorMessage = "アクセス権限がありません";
              break;
            case 404:
              errorMessage = "要求されたリソースが見つかりません";
              break;
            case 500:
              errorMessage = "サーバーエラーが発生しました";
              break;
            default:
              errorMessage = `エラー: ${error.status} - ${error.statusText}`;
          }
        }

        // エラーメッセージを表示
        this.messageService.addError(errorMessage);

        return throwError(() => error);
      })
    );
  }
}
```

### Step 11: コンポーネントでの使用例

```typescript
// 特定のコンポーネント内での使用例
import { Component, inject } from "@angular/core";
import { MessageService } from "../../services/message.service";

@Component({
  selector: "app-user-profile",
  template: `
    <div class="profile-container">
      <h2>ユーザープロフィール</h2>

      <button (click)="saveProfile()">プロフィールを保存</button>
      <button (click)="deleteProfile()">アカウントを削除</button>
    </div>
  `,
})
export class UserProfileComponent {
  private messageService = inject(MessageService);

  saveProfile(): void {
    // プロフィール保存ロジック
    // ...

    this.messageService.addSuccess("プロフィールが正常に保存されました");
  }

  deleteProfile(): void {
    this.messageService.addWarning("アカウントを削除しますか？", {
      autoClose: false,
      action: {
        text: "削除を確認",
        handler: () => this.confirmDelete(),
      },
    });
  }

  confirmDelete(): void {
    // 削除ロジック
    // ...

    this.messageService.addInfo(
      "アカウントが削除されました。ログアウトします..."
    );
  }
}
```

### Step 12: メッセージ履歴の実装

```typescript
// services/message.service.ts - 履歴機能の追加
private messageHistorySignal = signal<Message[]>([]);

public messageHistory = computed(() => this.messageHistorySignal());

// メッセージ削除時に履歴に追加
removeMessage(id: string): void {
  const message = this.messagesSignal().find(m => m.id === id);

  if (message) {
    // 履歴に追加
    this.messageHistorySignal.update(history => {
      // 最大20件まで保持
      const newHistory = [message, ...history];
      return newHistory.slice(0, 20);
    });
  }

  // メッセージリストから削除
  this.messagesSignal.update(messages =>
    messages.filter(m => m.id !== id)
  );
}
```

## ユースケース例

1. **フォーム送信後の確認メッセージ**

   ```typescript
   this.messageService.addSuccess("フォームが送信されました");
   ```

2. **ユーザーアクションが必要なメッセージ**

   ```typescript
   this.messageService.addWarning("セッションが間もなく終了します", {
     autoClose: false,
     action: {
       text: "セッションを延長",
       handler: () => this.extendSession(),
     },
   });
   ```

3. **エラーメッセージ**

   ```typescript
   this.messageService.addError(
     "データの読み込み中にエラーが発生しました。再読み込みしてください。"
   );
   ```

4. **リダイレクト前の通知**

   ```typescript
   const msgId = this.messageService.addInfo(
     "ダッシュボードにリダイレクトします...",
     {
       duration: 2000,
     }
   );

   setTimeout(() => {
     this.router.navigate(["/dashboard"]);
   }, 2000);
   ```

## ベストプラクティス

1. **簡潔なメッセージ** - ユーザーが素早く理解できる短いメッセージを使用する
2. **適切なメッセージタイプ** - 情報の重要度や緊急性に合わせたタイプを選ぶ
3. **自動閉じる時間の調整** - メッセージの重要度や長さに応じて表示時間を調整する
4. **アクション付きメッセージの活用** - ユーザーに選択肢やアクションを提供する
5. **アクセシビリティ対応** - 色だけでなくアイコンや構造でメッセージタイプを識別できるようにする

以上のステップに従うことで、Angular アプリケーションにおける直感的でユーザーフレンドリーなメッセージングシステムが実装できます。Signals を活用した実装により、効率的なリアクティブな状態管理が可能になります。
