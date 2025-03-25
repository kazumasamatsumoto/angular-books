# Angular Signals と不変性 - オブジェクトの取り扱い

Angular Signals を使用する際のオブジェクトの不変性（Immutability）は非常に重要な概念です。以下にその要点をまとめます。

## シグナルとオブジェクトの不変性

```typescript
import { signal } from "@angular/core";

// オブジェクト型のシグナルを作成
const user = signal({ name: "Alice", age: 30 });
```

### 不変性の原則

Angular Signals では、オブジェクトを変更する場合は新しいオブジェクトインスタンスを作成する必要があります。これが不変性（Immutability）の原則です。

```typescript
// ❌ 誤った方法: オブジェクトを直接変更
user().name = "Bob"; // これはシグナルの変更を検知できない！

// ✅ 正しい方法: 新しいオブジェクトを作成して設定
user.set({ name: "Bob", age: 30 });
```

### update メソッドを使用した安全な更新

```typescript
// オブジェクトの一部のプロパティのみを更新
user.update((currentUser) => ({
  ...currentUser, // 既存のプロパティをスプレッド構文でコピー
  name: "Bob", // 更新したいプロパティのみ上書き
}));
```

## ネストされたオブジェクトの取り扱い

複雑なネストされたオブジェクトを扱う場合も、更新する際は完全に新しいオブジェクトを作成します。

```typescript
const profile = signal({
  user: {
    name: "Alice",
    contact: {
      email: "alice@example.com",
      phone: "123-456-7890",
    },
  },
});

// ネストされたプロパティを更新する場合
profile.update((current) => ({
  ...current,
  user: {
    ...current.user,
    contact: {
      ...current.user.contact,
      email: "newalice@example.com",
    },
  },
}));
```

## イミュータビリティのヘルパーライブラリ

深くネストされたオブジェクトを扱う場合、スプレッド構文だけでは煩雑になることがあります。このような場合は以下のライブラリが役立ちます：

- **Immer**: シンプルな構文で不変更新ができる
- **immutable.js**: 不変データ構造を提供する

```typescript
// Immerの例
import { produce } from "immer";

profile.update((current) =>
  produce(current, (draft) => {
    draft.user.contact.email = "newalice@example.com";
  })
);
```

## パフォーマンスの考慮事項

1. **浅い比較**: Angular Signals はオブジェクトの参照が変わったかどうかを確認するだけなので、参照が変わらなければ変更を検知できません。

2. **大きなオブジェクト**: 巨大なオブジェクトを毎回コピーするのは非効率なので、適切な粒度でシグナルを分割することを検討しましょう。

```typescript
// 良い例: 関連するデータを別々のシグナルに分割する
const userData = signal({ name: "Alice", age: 30 });
const userContacts = signal({
  email: "alice@example.com",
  phone: "123-456-7890",
});
```

## まとめ

- シグナルのオブジェクト値を変更する際は必ず新しいオブジェクトを作成する
- `set()`や`update()`メソッドを使用して更新する
- スプレッド構文や不変性のヘルパーライブラリを活用する
- 必要に応じてシグナルを適切な粒度に分割する

これらの原則に従うことで、Angular Signals のリアクティブ機能を最大限に活用し、予測可能なデータフローを実現できます。
