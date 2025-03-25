# Angular Signals と不変性 - 配列の取り扱い

Angular Signals で配列を扱う場合も、オブジェクトと同様に不変性（Immutability）の原則が適用されます。以下に配列特有の扱い方をまとめます。

## 基本: 配列型シグナルの作成

```typescript
import { signal } from "@angular/core";

// 配列型のシグナルを作成
const todos = signal([
  { id: 1, text: "Learn Angular", completed: false },
  { id: 2, text: "Learn Signals", completed: false },
]);
```

## 配列更新の正しい方法

### 誤った方法 ❌

```typescript
// 直接配列を変更 - これはシグナルの変更を検知できない
todos().push({ id: 3, text: "Learn RxJS", completed: false });
todos()[0].completed = true;
```

### 正しい方法 ✅

```typescript
// 1. 新しい配列を作成して設定
todos.set([...todos(), { id: 3, text: "Learn RxJS", completed: false }]);

// 2. update メソッドを使用
todos.update((currentTodos) => [
  ...currentTodos,
  { id: 3, text: "Learn RxJS", completed: false },
]);
```

## 一般的な配列操作

### 要素の追加

```typescript
// 末尾に追加
todos.update((current) => [...current, newTodo]);

// 先頭に追加
todos.update((current) => [newTodo, ...current]);

// 特定の位置に追加
todos.update((current) => [
  ...current.slice(0, index),
  newTodo,
  ...current.slice(index),
]);
```

### 要素の削除

```typescript
// ID に基づく削除
todos.update((current) => current.filter((todo) => todo.id !== idToRemove));

// インデックスに基づく削除
todos.update((current) => [
  ...current.slice(0, indexToRemove),
  ...current.slice(indexToRemove + 1),
]);
```

### 要素の更新

```typescript
// 特定の要素を更新
todos.update((current) =>
  current.map((todo) =>
    todo.id === todoId ? { ...todo, completed: !todo.completed } : todo
  )
);
```

### 並べ替え

```typescript
// アルファベット順に並べ替え
todos.update((current) =>
  [...current].sort((a, b) => a.text.localeCompare(b.text))
);
```

## パフォーマンスを考慮した実装

大きな配列を扱う場合は、毎回全体をコピーするのではなく、必要な部分だけを更新するアプローチを検討します。

```typescript
// 大きな配列の特定のインデックスだけを更新
todos.update((current) => {
  const newArray = [...current];
  newArray[index] = { ...newArray[index], completed: true };
  return newArray;
});
```

## イミュータブルな配列操作のヘルパー関数

複雑な操作を単純化するために、ヘルパー関数を作成すると便利です。

```typescript
function addTodo(todos, newTodo) {
  return [...todos, newTodo];
}

function removeTodo(todos, id) {
  return todos.filter((todo) => todo.id !== id);
}

function updateTodo(todos, id, changes) {
  return todos.map((todo) => (todo.id === id ? { ...todo, ...changes } : todo));
}

// 使用例
todos.update((current) => addTodo(current, newTodo));
```

## まとめ

1. 配列を変更する際は、必ず新しい配列インスタンスを作成する
2. `push()`, `pop()`, `splice()` などの変更メソッドは直接使用しない
3. スプレッド構文、`map()`, `filter()` などの非変更メソッドを活用する
4. 大きな配列では必要に応じて最適化を検討する
5. 共通操作はヘルパー関数にまとめると便利

これらの原則に従うことで、Angular Signals の変更検知メカニズムを正しく機能させ、予測可能なデータフローを維持できます。
