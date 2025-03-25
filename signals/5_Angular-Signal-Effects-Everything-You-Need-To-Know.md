# Angular Signal Effects - すべてを知る

Angular Signal Effects は、Angular のシグナルシステムを拡張し、リアクティブな副作用を処理するための強力な機能です。シグナルの変化に応じて自動的に実行されるコードブロックを定義できます。

## Signal Effects とは

Signal Effects は、一つ以上のシグナルの値が変更されたときに自動的に実行される関数です。これにより、シグナルの変化に基づいた副作用（DOM 更新、API 呼び出し、ログ記録など）を宣言的に定義できます。

```typescript
import { effect, signal } from "@angular/core";

const count = signal(0);

// シグナルの変更を監視するエフェクトを作成
effect(() => {
  console.log(`Count changed: ${count()}`);
  // countシグナルが変更されるたびにこのコードが実行される
});

// 値を更新するとエフェクトが発火
count.set(1); // Console: "Count changed: 1"
count.update((value) => value + 1); // Console: "Count changed: 2"
```

## 主な特徴

1. **自動依存関係追跡**: エフェクト内で参照されるすべてのシグナルが自動的に追跡され、それらが変更されるとエフェクトが再実行されます。

2. **クリーンアップ関数**: エフェクト関数からクリーンアップ関数を返すことで、リソースの解放やイベントリスナーの削除などを行えます。

3. **依存関係の動的変更**: エフェクト実行中に参照されるシグナルは実行ごとに再評価されるため、条件付きの依存関係も自然に処理できます。

4. **コンポーネントライフサイクルとの統合**: コンポーネント内で使用すると、コンポーネントの破棄時に自動的にエフェクトも破棄されます。

## Signal Effects の使用例

### 1. DOM 操作

```typescript
@Component({
  selector: "app-counter",
  template: `<button (click)="increment()">Increment</button>`,
})
export class CounterComponent {
  count = signal(0);

  constructor() {
    effect(() => {
      document.title = `Count: ${this.count()}`;
      // countが変更されるたびにページタイトルが更新される
    });
  }

  increment() {
    this.count.update((n) => n + 1);
  }
}
```

### 2. API リクエスト

```typescript
@Component({
  selector: "app-user-profile",
  template: `
    <input [(ngModel)]="searchTerm()" />
    <div *ngIf="loading()">Loading...</div>
    <div *ngIf="user()">{{ user().name }}</div>
  `,
})
export class UserProfileComponent {
  searchTerm = signal("");
  user = signal(null);
  loading = signal(false);

  constructor(private http: HttpClient) {
    // 検索語が変更されたら新しいユーザーを取得
    effect(() => {
      const term = this.searchTerm();
      if (!term) return;

      this.loading.set(true);
      // HTTP呼び出しへのサブスクリプションをクリーンアップ
      const subscription = this.http
        .get(`/api/users?q=${term}`)
        .subscribe((user) => {
          this.user.set(user);
          this.loading.set(false);
        });

      // クリーンアップ関数を返す
      return () => subscription.unsubscribe();
    });
  }
}
```

### 3. ローカルストレージとの同期

```typescript
@Component({
  selector: "app-preferences",
  template: `
    <label>
      Dark Mode:
      <input
        type="checkbox"
        [checked]="darkMode()"
        (change)="toggleDarkMode()"
      />
    </label>
  `,
})
export class PreferencesComponent {
  darkMode = signal(JSON.parse(localStorage.getItem("darkMode") || "false"));

  constructor() {
    effect(() => {
      // darkModeシグナルが変更されるたびにローカルストレージを更新
      localStorage.setItem("darkMode", JSON.stringify(this.darkMode()));

      // テーマクラスを更新
      if (this.darkMode()) {
        document.body.classList.add("dark-theme");
      } else {
        document.body.classList.remove("dark-theme");
      }
    });
  }

  toggleDarkMode() {
    this.darkMode.update((current) => !current);
  }
}
```

## 注意点とベストプラクティス

1. **無限ループに注意**: エフェクト内でシグナルの値を変更すると、そのシグナルに依存する他のエフェクトが再実行され、無限ループが発生する可能性があります。

```typescript
// 無限ループの例
const count = signal(0);

effect(() => {
  console.log(count());
  count.update((c) => c + 1); // 危険: 同じエフェクト内でシグナルを更新
});
```

2. **エフェクトの分離**: 各エフェクトは一つの目的に集中させ、複数の関連しない副作用を一つのエフェクトに詰め込まないようにしましょう。

3. **非同期処理には注意**: エフェクト内で非同期処理を行う場合は、クリーンアップ関数で適切にリソースを解放することが重要です。

4. **パフォーマンスへの配慮**: 頻繁に変更されるシグナルに対するエフェクトでは、重い処理や頻繁な DOM 操作を避けましょう。

## 手動でのエフェクト破棄

エフェクトはコンポーネントのライフサイクルに合わせて自動的に破棄されますが、必要に応じて手動で破棄することもできます。

```typescript
import { effect } from "@angular/core";

const count = signal(0);
const dispose = effect(() => {
  console.log(`Count: ${count()}`);
});

// エフェクトを手動で破棄
dispose();

// 破棄後はシグナルが変更されてもエフェクトは実行されない
count.set(1); // コンソールには何も出力されない
```

## まとめ

Angular Signal Effects は、シグナルの変更に応じて副作用を宣言的に定義するための強力な機能です。DOM 操作、API リクエスト、状態の同期など、様々なユースケースに対応できます。適切に使用することで、コンポーネントのリアクティブな動作を効率的に実装でき、アプリケーションの保守性と可読性が向上します。
