# Angular Injectable Decorator

`@Injectable()` デコレータは Angular の依存性注入（DI）システムの中核となる要素です。このデコレータを使うことで、クラスが Angular の依存性注入システムによって管理されることを示します。

## 基本的な使い方

```typescript
import { Injectable } from "@angular/core";

@Injectable({
  providedIn: "root",
})
export class MyService {
  constructor() {}

  getData() {
    return "Some data from service";
  }
}
```

## @Injectable() デコレータの主な特徴

1. **サービスの登録**: クラスを DI システムに登録し、インジェクション可能にします

2. **providedIn オプション**:

   - `'root'`: アプリケーション全体で単一のインスタンスを共有（シングルトン）
   - `'platform'`: 複数のアプリで共有される場合に使用
   - `'any'`: 遅延ロードモジュールごとに新しいインスタンスを作成
   - 特定のモジュール: `providedIn: SomeModule` のように指定可能

3. **シングルトンの自動作成**: `providedIn: 'root'` を指定すると、アプリケーション起動時にサービスのシングルトンインスタンスが自動的に作成されます

## サービスの注入方法

Injectable なサービスは、他のクラスのコンストラクタで注入できます：

```typescript
import { Component } from "@angular/core";
import { MyService } from "./my.service";

@Component({
  selector: "app-root",
  template: `<h1>{{ title }}</h1>`,
})
export class AppComponent {
  title: string;

  constructor(private myService: MyService) {
    this.title = this.myService.getData();
  }
}
```

## ツリーシェイキング対応

`providedIn` 構文は「ツリーシェイキング」に対応しており、サービスが使用されない場合は最終的なバンドルから除外されます。これによりアプリケーションのサイズを最適化できます。

## 従来の方法との比較

以前の Angular バージョンや別の方法では、NgModule の providers 配列にサービスを登録していました：

```typescript
@NgModule({
  providers: [MyService],
})
export class AppModule {}
```

この方法でも機能しますが、`providedIn` の使用が推奨されるようになりました。

## 依存関係を持つサービス

Injectable サービスは他のサービスに依存することもできます：

```typescript
@Injectable({
  providedIn: "root",
})
export class DataService {
  constructor(private http: HttpClient) {}

  fetchData() {
    return this.http.get("/api/data");
  }
}
```

以上が Angular の Injectable デコレータの基本的な概要です。依存性注入は Angular アプリケーションの設計において重要な役割を果たし、テスト可能でモジュール化されたコードの作成を可能にします。
