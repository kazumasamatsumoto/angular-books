# Angular Signal Effects: マニュアルエフェクトクリーンアップの方法

Angular Signal Effects を使用する際、エフェクトのクリーンアップ（破棄）を適切に管理することは重要です。ここでは、エフェクトを手動でクリーンアップする方法について説明します。

## 基本的なエフェクトのクリーンアップ

エフェクトを作成すると、そのエフェクトを破棄するための関数（ディスポーザー）が返されます：

```typescript
import { effect } from "@angular/core";

// エフェクトを作成し、ディスポーザー関数を取得
const dispose = effect(() => {
  console.log("Effect running");
});

// 後でエフェクトを手動で破棄
dispose();
```

## クリーンアップが必要な状況

1. **コンポーネントライフサイクル外でのエフェクト**：コンポーネント内で作成されたエフェクトは通常、コンポーネントが破棄されると自動的にクリーンアップされますが、グローバルサービスやコンポーネント外で作成されたエフェクトは手動でクリーンアップする必要があります。

2. **動的に作成されるエフェクト**：条件に基づいて作成/破棄が必要なエフェクト。

3. **メモリリーク防止**：長時間実行されるアプリケーションでは、不要になったエフェクトを破棄してメモリリークを防ぐことが重要です。

## 実装例

### 1. サービス内でのエフェクトクリーンアップ

```typescript
@Injectable({
  providedIn: "root",
})
export class DataSyncService implements OnDestroy {
  private counterSignal = signal(0);
  private effectRef: (() => void) | null = null;

  startSync() {
    // 既存のエフェクトがあれば破棄
    this.stopSync();

    // 新しいエフェクトを作成
    this.effectRef = effect(() => {
      console.log(`Syncing data: ${this.counterSignal()}`);
      // データ同期処理...
    });
  }

  stopSync() {
    if (this.effectRef) {
      // エフェクトをクリーンアップ
      this.effectRef();
      this.effectRef = null;
    }
  }

  ngOnDestroy() {
    // サービスが破棄されるときにエフェクトをクリーンアップ
    this.stopSync();
  }
}
```

### 2. EffectRef を使用したクリーンアップ

Angular 16.1 以降では、`EffectRef`インターフェースが導入され、より直感的な API でエフェクトを管理できます：

```typescript
import { effect, EffectRef } from '@angular/core';

@Component({...})
export class FeatureComponent implements OnDestroy {
  private effectRef: EffectRef | null = null;

  startFeature() {
    // 既存のエフェクトがあれば破棄
    this.stopFeature();

    // EffectRefを取得
    this.effectRef = effect(() => {
      // エフェクトの内容...
    });
  }

  stopFeature() {
    // destroy()メソッドでエフェクトをクリーンアップ
    this.effectRef?.destroy();
    this.effectRef = null;
  }

  ngOnDestroy() {
    this.stopFeature();
  }
}
```

### 3. DestroyRef との統合

Angular 16 以降では、`DestroyRef`を使用してコンポーネントやディレクティブの破棄時に自動的にクリーンアップするコードを登録できます：

```typescript
@Component({...})
export class AdvancedComponent {
  constructor() {
    const destroyRef = inject(DestroyRef);

    // エフェクトの外部で作成されるリソース
    const interval = setInterval(() => {
      // 何らかの処理...
    }, 1000);

    // コンポーネントが破棄されたときにインターバルをクリア
    destroyRef.onDestroy(() => {
      clearInterval(interval);
    });

    // エフェクトも登録可能
    const disposeEffect = effect(() => {
      // エフェクトの内容...
    });

    // エフェクトのクリーンアップも登録
    destroyRef.onDestroy(() => {
      disposeEffect();
    });
  }
}
```

## エフェクト内部でのクリーンアップ

エフェクト関数自体からクリーンアップ関数を返すことで、エフェクトが再実行される前やエフェクトが破棄されるときに実行される処理を定義できます：

```typescript
effect(() => {
  const subscription = someObservable.subscribe();

  // このクリーンアップ関数はエフェクトが再実行される前や
  // エフェクトが破棄されるときに呼び出される
  return () => {
    subscription.unsubscribe();
  };
});
```

## 注意点

1. **複数のエフェクトの管理**: 多数のエフェクトを管理する場合は、配列などを使って参照を保持し、一括でクリーンアップできるようにします。

2. **安全なクリーンアップ**: クリーンアップする前に必ずエフェクト参照が null でないことを確認します。

3. **循環参照の防止**: エフェクトのクリーンアップ関数内でエフェクト自体を参照しないよう注意します。

4. **コンポーネントのライフサイクル**: `ngOnDestroy`内でエフェクトをクリーンアップすることを忘れないようにします。

## まとめ

Angular Signal Effects の手動クリーンアップは、特にコンポーネントのライフサイクル外で作成されたエフェクトや、動的に作成されるエフェクトを管理する際に重要です。適切なクリーンアップを行うことで、メモリリークを防ぎ、アプリケーションのパフォーマンスを向上させることができます。

はい、その通りです。エフェクトをクリーンアップ（破棄）するということは、そのエフェクトの効果を停止させることを意味します。

具体的には：

1. そのエフェクトは、関連するシグナルの値が変更されても、もう実行されなくなります。
2. エフェクト内で設定していた監視や副作用（DOM の更新、API の呼び出し、データの計算など）が行われなくなります。
3. エフェクト内部で返していたクリーンアップ関数が実行され、リソース（サブスクリプション、タイマー、イベントリスナーなど）が解放されます。

例えば：

```typescript
// エフェクトを作成
const count = signal(0);
const disposeEffect = effect(() => {
  console.log(`現在のカウント: ${count()}`);
  document.title = `カウント: ${count()}`;
});

// カウントを更新すると、エフェクトが実行される
count.set(1); // ログに「現在のカウント: 1」と表示され、タイトルも更新される

// エフェクトをクリーンアップ（破棄）
disposeEffect();

// クリーンアップ後はカウントを更新しても、エフェクトは実行されない
count.set(2); // ログに何も表示されず、タイトルも更新されない
```

これは、コンポーネントが画面から削除されたときや、特定の条件下でエフェクトが不要になったときなど、リソースの解放や不要な処理の停止のために重要です。
