# A Crash Course

さっそく始めてみましょう! Vue Test Utils (VTU) を学ぶために、簡単な Todo アプリを作り、テストを書きながら進めていきましょう。
このガイドでは、以下の方法について説明します:

- Mount components: コンポーネントをマウントする
- Find elements: 要素を見つける
- Fill out forms: フォームに入力する
- Trigger events: イベントのトリガー

## Getting Started

まずは、Todo を 1 つ持つシンプルな `TodoApp` コンポーネントを作成します:

```vue
<template>
  <div></div>
</template>

<script>
export default {
  name: 'TodoApp',

  data() {
    return {
      todos: [
        {
          id: 1,
          text: 'Learn Vue.js 3',
          completed: false
        }
      ]
    }
  }
}
</script>
```

## 最初のテスト - Todo のレンダリング {#the-first-test-a-todo-is-rendered}

最初のテストは、Todo がレンダリングされたことを確認するものです。まずこのテストを見てから、各パーツについて説明します:

```js
import { mount } from '@vue/test-utils'
import TodoApp from './TodoApp.vue'

test('renders a todo', () => {
  const wrapper = mount(TodoApp)

  const todo = wrapper.get('[data-test="todo"]')

  expect(todo.text()).toBe('Learn Vue.js 3')
})
```

まず、 `mount` をインポートすることから始めます。これは、VTU でコンポーネントをレンダリングするための主な方法です。テストの短い説明と一緒に `test` 関数を使用してテストを宣言します。`test` 関数と `expect` 関数は、ほとんどのテストランナーでグローバルに利用可能です（この例では [Jest](https://jestjs.io/en/) を使用しています）。もし `test` と `expect` が分かりにくそうなら、Jest のドキュメントに [その使い方と動作の簡単な例](https://jestjs.io/docs/en/getting-started) があります。

次に、`mount` を呼び出し、第一引数としてコンポーネントを渡します。これは、あなたが書くほとんどすべてのテストが行うことです。`mount` は、テストに便利ないくつかのメソッドで、アプリの周りにシンプルな「ラッパー」を提供するので、結果は慣習的に `wrapper` と呼ばれる変数に代入されます。

最後に、多くのテストランナー（Jest も含む）に共通するもうひとつのグローバル関数 `expect` を使用します。これは、実際の出力が私たちの考えるものと一致することを保証する、あるいは _期待する_ というものです。この場合、`data-test="todo"` というセレクタを持つ要素を探します。DOM では `<div data-test="todo">...</div>` のようになります。次に、`text` メソッドを呼び出してコンテンツを取得します。コンテンツは `'Learn Vue.js 3'` であると予想されます。

> `data-test` セレクタを使用することは必須ではありませんが、テストをよりもろくすることができます。 クラスや id はアプリケーションの成長とともに変更されたり移動したりする傾向があります。

## テストをパスさせる {#making-the-test-pass}

今、このテストを実行すると、次のようなエラーメッセージが出て失敗します: `Unable to get [data-test="todo"]`。これは、Todo 項目をレンダリングしていないので、`get()` 呼び出しがラッパーを返すのに失敗しているからです（ VTU はすべてのコンポーネント、および DOM 要素を、いくつかの便利なメソッドを持つ「ラッパー」でラップすることを思い出してください）。`TodoApp.vue` の `<template>` を更新して、`todos`の配列をレンダリングしてみましょう:

```vue
<template>
  <div>
    <div v-for="todo in todos" :key="todo.id" data-test="todo">
      {{ todo.text }}
    </div>
  </div>
</template>
```

この変更で、テストは合格です。おめでとうございます! あなたは最初のコンポーネントテストを書きました。

## 新しい Todo を追加する {#adding-a-new-todo}

次に追加する機能は、ユーザーが新しい Todo を作成できるようにすることです。そのためには、ユーザーがテキストを入力するための入力フォームが必要です。ユーザーがフォームを送信すると、新しい Todo がレンダリングされることを期待します。では、テストを見てみましょう:

```js
import { mount } from '@vue/test-utils'
import TodoApp from './TodoApp.vue'

test('creates a todo', () => {
  const wrapper = mount(TodoApp)
  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(1)

  wrapper.get('[data-test="new-todo"]').setValue('New todo')
  wrapper.get('[data-test="form"]').trigger('submit')

  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(2)
})
```

いつものように、まず `mount` を使って要素をレンダリングしています。また、1 つの Todo だけがレンダリングされることを表明しています。これにより、テストの最終行が示すように、追加のTodoを追加していることが明らかになります。

`<input>` を更新するには、`setValue` を使用します - これで入力の値を設定できます。

`<input>` を更新した後、`trigger` メソッドを使用して、ユーザーがフォームを送信したことをシミュレートしています。最後に、Todo アイテムの数が 1 個から 2 個に増えたことを表明します。

このテストを実行すると、明らかに失敗します。`TodoApp.vue` を更新して `<form>` と `<input>` 要素を持たせ、テストが通るようにしましょう:

```vue
<template>
  <div>
    <div v-for="todo in todos" :key="todo.id" data-test="todo">
      {{ todo.text }}
    </div>

    <form data-test="form" @submit.prevent="createTodo">
      <input data-test="new-todo" v-model="newTodo" />
    </form>
  </div>
</template>

<script>
export default {
  name: 'TodoApp',

  data() {
    return {
      newTodo: '',
      todos: [
        {
          id: 1,
          text: 'Learn Vue.js 3',
          completed: false
        }
      ]
    }
  },

  methods: {
    createTodo() {
      this.todos.push({
        id: 2,
        text: this.newTodo,
        completed: false
      })
    }
  }
}
</script>
```

`v-model` を使って `<input>` にバインドし、`@submit` を使ってフォームの送信を待ち受けます。フォームが送信されると、`createTodo` が呼び出され、新しい Todo が `todos` 配列に挿入されます。

これは良いように見えますが、テストを実行するとエラーが表示されます:

```
expect(received).toHaveLength(expected)

    Expected length: 2
    Received length: 1
    Received array:  [{"element": <div data-test="todo">Learn Vue.js 3</div>}]
```

todos は増えていません。問題は、Jest がテストを同期的に実行し、最後の関数が呼ばれると同時にテストを終了してしまうことです。しかし、Vue は DOM を非同期で更新します。テストを `async` でマークし、DOM を変更させる可能性のあるメソッドに対して `await` を呼び出す必要があります。`trigger` はそのようなメソッドの一つで、`setValue` も同様です。単に `await` を前に置くだけで、テストは期待通りに動作するはずです。

```js
import { mount } from '@vue/test-utils'
import TodoApp from './TodoApp.vue'

test('creates a todo', async () => {
  const wrapper = mount(TodoApp)

  await wrapper.get('[data-test="new-todo"]').setValue('New todo')
  await wrapper.get('[data-test="form"]').trigger('submit')

  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(2)
})
```

これで、いよいよテストも合格です!

## Todo を完了させる {#completing-a-todo}

Todo を作成できるようになったので、ユーザーにチェックボックスで Todo 項目を完了/未完了にする機能を与えてみましょう。前回と同様に、失敗するテストから始めましょう:

```js
import { mount } from '@vue/test-utils'
import TodoApp from './TodoApp.vue'

test('completes a todo', async () => {
  const wrapper = mount(TodoApp)

  await wrapper.get('[data-test="todo-checkbox"]').setValue(true)

  expect(wrapper.get('[data-test="todo"]').classes()).toContain('completed')
})
```

このテストは前の 2 つと似ています。要素を見つけて、同じように操作します（`<input>` と操作しているので、再び `setValue` を使用します）。

最後に、アサーションを行います。完了した ToDo には `completed` クラスを適用します。これを利用して、Todo の状態を視覚的に示すスタイルを追加します。

`<template>` を更新して、`<input type="checkbox">` と todo 要素のクラスバインディングを含めることで、このテストに合格するようにできます:

```vue
<template>
  <div>
    <div
      v-for="todo in todos"
      :key="todo.id"
      data-test="todo"
      :class="[todo.completed ? 'completed' : '']"
    >
      {{ todo.text }}
      <input
        type="checkbox"
        v-model="todo.completed"
        data-test="todo-checkbox"
      />
    </div>

    <form data-test="form" @submit.prevent="createTodo">
      <input data-test="new-todo" v-model="newTodo" />
    </form>
  </div>
</template>
```

おめでとうございます! あなたは最初のコンポーネントテストを書きました。

## Arrange, Act, Assert

それぞれのテストにおいて、コードの間に新しい行があることにお気づきでしょうか。もう一度、2 番目のテストを詳しく見てみましょう:

```js
import { mount } from '@vue/test-utils'
import TodoApp from './TodoApp.vue'

test('creates a todo', async () => {
  const wrapper = mount(TodoApp)

  await wrapper.get('[data-test="new-todo"]').setValue('New todo')
  await wrapper.get('[data-test="form"]').trigger('submit')

  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(2)
})
```

テストは 3 つのステージに分けられ、改行で区切られています。この 3 つのステージは、テストの 3 つの段階、すなわち **アレンジ**、**アクション**、**アサート** を表しています。

_アレンジ_ 段階では、テスト用のシナリオを設定します。より複雑な例では、Vuex ストアを作成したり、データベースにデータを入力したりする必要があるかもしれません。

_アクション_ フェーズでは、ユーザーがコンポーネントやアプリケーションをどのように操作するかをシミュレートし、シナリオを実行します。

_アサート_ フェーズでは、コンポーネントの現在の状態がどのようなものであるべきかを表明します。

ほとんどすべてのテストは、この 3 つのフェーズに従うことになります。このガイドのように改行で区切る必要はありませんが、 テストを書く際にはこの 3 つのフェーズを念頭に置いておくとよいでしょう。

## 結論 {#conclusion}

- `mount()` を使用して、コンポーネントをレンダリングします。
- DOM を照会するには、`get()` および `findAll()` を使用します。
- `trigger()` と `setValue()` は、ユーザー入力をシミュレートするためのヘルパーです。
- DOM の更新は非同期処理なので、`async` と `await` を必ず使用します。
- テストは通常、arrange, act, assert の 3 つのフェーズで構成されます。
