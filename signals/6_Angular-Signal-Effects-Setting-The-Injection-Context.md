# Angular Signal Effects - インジェクションコンテキストの設定

Angular Signal Effects のインジェクションコンテキスト設定は、エフェクト内で依存性注入を正しく機能させるための重要な概念です。これによりエフェクト内で Angular の DI システムを活用できるようになります。

## インジェクションコンテキストとは

エフェクトは Angular の依存性注入（DI）システムの外で実行される可能性があるため、デフォルトではサービスなどの注入された依存関係にアクセスできません。インジェクションコンテキストを設定することで、エフェクトが特定のコンポーネントやディレクティブのコンテキスト内で実行されるように指示できます。

## インジェクションコンテキストの設定方法

`effect()`関数の第 2 引数にオプションオブジェクトを渡し、`injector`プロパティを設定します：

```typescript
import { effect, Injector, inject } from "@angular/core";

@Component({
  selector: "app-example",
  template: "...",
})
export class ExampleComponent {
  private myService = inject(MyService);

  constructor() {
    // コンポーネントのインジェクターを使用
    effect(
      () => {
        // ここでmyServiceを使用できる
        this.myService.doSomething();
      },
      { injector: inject(Injector) }
    );
  }
}
```

## 主なユースケース

### 1. コンポーネント内でのサービス利用

```typescript
@Component({
  selector: "app-data-component",
  template: `<div>{{ data() }}</div>`,
})
export class DataComponent {
  private dataService = inject(DataService);
  data = signal(null);

  constructor() {
    effect(
      () => {
        // 現在のURLパラメータに基づいてデータを取得
        const id = this.route.snapshot.params["id"];
        this.dataService.fetchData(id).subscribe((result) => {
          this.data.set(result);
        });
      },
      { injector: inject(Injector) }
    );
  }
}
```

### 2. ディレクティブでのエフェクト

```typescript
@Directive({
  selector: "[appHighlight]",
})
export class HighlightDirective {
  @Input() highlightColor = signal("yellow");

  constructor(private el: ElementRef) {
    effect(
      () => {
        // 色が変わるたびに要素のスタイルを更新
        this.el.nativeElement.style.backgroundColor = this.highlightColor();
      },
      { injector: inject(Injector) }
    );
  }
}
```

### 3. スタンドアロンエフェクトでの DI

```typescript
// スタンドアロンのユーティリティ関数
export function createLogger(source: string) {
  return effect(() => {
    const loggingService = inject(LoggingService);
    const authService = inject(AuthService);

    // 認証されたユーザーのアクションのみをログに記録
    if (authService.isAuthenticated()) {
      loggingService.log(`[${source}] Action performed by ${authService.getCurrentUser()}`);
    }
  }, { injector: inject(Injector) });
}

// 使用例
@Component({...})
export class SomeComponent {
  constructor() {
    createLogger('SomeComponent');
  }
}
```

## 高度なテクニック

### カスタムインジェクターの提供

特定のプロバイダーを持つカスタムインジェクターを作成してエフェクトに渡すことができます：

```typescript
import { Injector, createInjector } from '@angular/core';

@Component({...})
export class AdvancedComponent {
  constructor() {
    // カスタムプロバイダーを持つインジェクター
    const customInjector = Injector.create({
      providers: [
        { provide: CONFIG_TOKEN, useValue: { debug: true } }
      ],
      parent: inject(Injector) // 親インジェクターを設定
    });

    effect(() => {
      const config = inject(CONFIG_TOKEN);
      if (config.debug) {
        console.log('Debug mode is enabled');
      }
    }, { injector: customInjector });
  }
}
```

### 動的インジェクションコンテキスト

インジェクションコンテキストを動的に変更することもできます：

```typescript
@Component({...})
export class DynamicContextComponent implements OnInit, OnDestroy {
  private readonly destroyRef = inject(DestroyRef);
  private effectRef: EffectRef | null = null;

  ngOnInit() {
    this.createContextualEffect();
  }

  private createContextualEffect() {
    // 既存のエフェクトをクリーンアップ
    this.effectRef?.destroy();

    this.effectRef = effect(() => {
      // 新しいコンテキストでのエフェクト
    }, { injector: inject(Injector) });

    // コンポーネント破棄時にエフェクトを破棄
    this.destroyRef.onDestroy(() => {
      this.effectRef?.destroy();
    });
  }
}
```

## 注意点とベストプラクティス

1. **循環依存関係に注意**: エフェクト内でサービスを使用し、そのサービスが同じエフェクトを使用するコンポーネントを参照すると循環依存関係が発生する可能性があります。

2. **エフェクトのスコープ**: インジェクションコンテキストを設定したエフェクトは、そのコンテキスト（コンポーネントなど）のライフサイクルに従います。

3. **インジェクターの使い回し**: 同じコンポーネント内の複数のエフェクトで同じインジェクターを使用することで、パフォーマンスを最適化できます。

4. **テスト容易性**: インジェクションコンテキストを適切に設定することで、モックサービスを使ったエフェクトのテストが容易になります。

## まとめ

Angular Signal Effects のインジェクションコンテキスト設定は、エフェクト内で依存性注入を活用するための重要な機能です。これにより、サービスやその他の注入可能な依存関係をエフェクト内でシームレスに使用でき、リアクティブな副作用と Angular の DI システムを効果的に組み合わせることができます。適切なインジェクションコンテキストを設定することで、より堅牢で保守しやすいエフェクトを実装できます。

コンポーネントの初期化時（constructor 内や ngOnInit 内など）にエフェクトを作成すること自体は可能です。しかし、エフェクト内で`inject()`を使ってサービスなどを取得しようとすると問題が発生します。

Angular のエフェクトは通常の JavaScript 関数として実行されるため、デフォルトでは Angular の依存性注入（DI）システムのコンテキスト外で動作します。つまり、エフェクト関数の内部では、通常`inject()`関数を使ってサービスを取得することができません。

`{ injector: inject(Injector) }`を設定することで、エフェクト関数内でも依存性注入が機能するように「コンテキスト」を提供しています。これにより、エフェクト内で`inject()`を使ってサービスを取得できるようになります。

例えば：

```typescript
@Component({...})
class MyComponent {
  // これはコンポーネントレベルで注入されたサービス
  private dataService = inject(DataService);

  constructor() {
    // エフェクト内で別のサービスを使用したい
    effect(() => {
      // これはエフェクト内で注入されたサービス
      const logService = inject(LogService); // ← injectorを設定しないとここでエラー

      // dataServiceの変更を監視して、変更があればログを出力
      logService.log(`Data changed: ${this.dataService.getData()}`);
    }, { injector: inject(Injector) }); // ← これによりエフェクト内でもinject()が使える
  }
}
```

要するに、インジェクションコンテキストの設定は「エフェクトが Angular の DI システムにアクセスできるようにする」ための仕組みです。これによりエフェクト内でもサービスなどの依存関係を注入して使用できるようになります。
