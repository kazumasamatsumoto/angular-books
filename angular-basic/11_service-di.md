# Angular Injectable Services

Angular の依存性注入(DI)システムと Injectable Services は、アプリケーション開発の中核をなす重要な機能です。ここでは、サービスの基本概念から初期化方法、最新のベストプラクティスまでを詳しく解説します。

## サービスとは何か

サービスは、アプリケーション全体で再利用可能なロジックをカプセル化する TypeScript クラスです。データの取得・処理、複雑なビジネスロジック、コンポーネント間の通信などを担います。

```typescript
import { Injectable } from "@angular/core";

@Injectable({
  providedIn: "root",
})
export class DataService {
  getData() {
    return ["item1", "item2", "item3"];
  }
}
```

## @Injectable() デコレータ

`@Injectable()` デコレータはクラスをサービスとして識別し、依存性注入システムに登録します。このデコレータには提供スコープを定義する設定オブジェクトを渡せます。

### providedIn オプション

- **'root'**: アプリケーション全体でシングルトンとして提供（推奨）
- **'platform'**: 複数のアプリで共有される全プラットフォームレベル
- **'any'**: 各遅延ロードモジュールで独自のインスタンス
- **特定のモジュール**: 指定したモジュールのスコープで提供

```typescript
@Injectable({
  providedIn: 'root' // アプリケーション全体でシングルトン
})
```

## サービスの提供方法

### 1. ルートレベル（アプリケーション全体）

```typescript
@Injectable({
  providedIn: 'root'
})
```

### 2. モジュールレベル

```typescript
@NgModule({
  providers: [UserService],
})
export class AppModule {}
```

### 3. コンポーネントレベル

```typescript
@Component({
  selector: 'app-user',
  providers: [UserService] // このコンポーネントとその子でのみ利用可能
})
```

## サービスの注入と使用

サービスはコンストラクタ注入を通じて使用されます：

```typescript
@Component({
  selector: "app-user-list",
})
export class UserListComponent {
  users: User[] = [];

  constructor(private userService: UserService) {
    this.users = this.userService.getUsers();
  }
}
```

## 最新のベストプラクティス

### スタンドアロンコンポーネントでのサービス

Angular 14 以降、スタンドアロンコンポーネントでは以下のようにサービスを提供できます：

```typescript
@Component({
  standalone: true,
  imports: [...],
  providers: [SpecificService],
  // ...
})
```

### 遅延ロード時のサービスインスタンス管理

```typescript
@Injectable({
  providedIn: 'any' // 各遅延ロードモジュールに独自のインスタンス
})
```

### 環境設定に基づくサービスプロバイダー

```typescript
@NgModule({
  providers: [
    { provide: API_URL, useValue: environment.apiUrl },
    { provide: LogService, useClass: environment.production ? ProdLogService : DevLogService }
  ]
})
```

## サービスのライフサイクル

サービスは `ngOnDestroy` ライフサイクルフックを実装でき、特にコンポーネントスコープで提供された場合に重要です：

```typescript
@Injectable()
export class DataService implements OnDestroy {
  private subscription: Subscription;

  constructor(private http: HttpClient) {
    this.subscription = this.http.get("...").subscribe();
  }

  ngOnDestroy() {
    this.subscription.unsubscribe(); // リソースのクリーンアップ
  }
}
```

## 階層的注入システム

Angular の注入システムは階層的で、解決順序は：

1. コンポーネントレベル
2. 親コンポーネント
3. モジュールレベル
4. ルートレベル

この階層は、同じサービスの異なるインスタンスを特定のコンポーネントツリーに提供する柔軟性をもたらします。

## まとめ

Angular の依存性注入と Injectable サービスは、疎結合で保守性の高いアプリケーションを構築するための基盤です。`providedIn: 'root'`を使用したシングルトンサービスが推奨されていますが、特定のユースケースに応じて異なる提供方法を選択できます。

サービスを効果的に活用することで、コンポーネント間のロジック共有、データの一元管理、そして全体的なアプリケーションアーキテクチャの改善が可能になります。
